# Throttling study

## Summary

The purpose of Kubernetes is to orchestrate many, potentially big, applications that could be co-located on shared minions. Kubernetes’ current policy when starting PODs is to start them asap.

This can be sub-optimal for OLTP applications where many processes/threads are started. During normal operation, those processes/threads are mostly idling, waiting for the next transaction, but when they are starting, they are CPU and IO greedy.

Starting all those processes simultaneously can cause start-up time degradation due to context switching and I/O saturation. In that case, starting them in a controlled manner can improve the start-up time and its predictability.

This study proposes to implement a throttling mechanism to improve start-up time predictability. It covers different possible policies, their pros/cons and implications.

## Table of Contents

  * [Throttling study](#throttling-study)
    * [Summary](#summary)
    * [Problematic](#problematic)
      * [Running does not imply Ready](#running-does-not-imply-ready)
      * [Need for throttling](#need-for-throttling)
        * [1. concurrency between the <code>starting</code> POD and the <code>ready</code> ones which are running on the same machine](#1-concurrency-between-the-starting-pod-and-the-ready-ones-which-are-running-on-the-same-machine)
        * [2. concurrency between the <code>starting</code> containers and the other <code>starting</code> ones of the same POD](#2-concurrency-between-the-starting-containers-and-the-other-starting-ones-of-the-same-pod)
    * [Protect the <code>ready</code> PODs from the <code>starting</code> ones on the same machine](#protect-the-ready-pods-from-the-starting-ones-on-the-same-machine)
    * [Limit the number of processes that are starting at a given time](#limit-the-number-of-processes-that-are-starting-at-a-given-time)
      * [Readiness detection](#readiness-detection)
        * [`ready` notification](#ready-notification)
        * [`ready` polling](#ready-polling)
        * [Preferred solution](#preferred-solution)
      * [Throttling policy](#throttling-policy)
        * [limit the number of processes in the <code>starting</code> state](#limit-the-number-of-processes-in-the-starting-state)
        * [time throttling](#time-throttling)
        * [resource monitoring](#resource-monitoring)
        * [Composite policy](#composite-policy)
      * [Configuration example](#configuration-example)
    * [Dependency management proposal](#dependency-management-proposal)
      * [Cycle detection](#cycle-detection)
    * [Combining throttling with dependency management](#combining-throttling-with-dependency-management)
      * [Example of why dependency management needs to be combined with throttling](#example-of-why-dependency-management-needs-to-be-combined-with-throttling)
      * [Final picture](#final-picture)

## Problematic

We have applications that require many processes to be spawned on the same machine.

In order to port those applications to Kubernetes/OpenShift, we plan to have many containers (with only one process inside) in a POD.

### Running does not imply Ready

It is not because a process is started that it is ready to work and process traffic.

For example, we can have a process that starts its life by fetching some configuration elements from somewhere and doing some expensive configuration stuff before being really ready to work.

Further in this document, we’ll make a distinction between `starting` and `ready` states.
* `starting` means that the container is running, the process is live, but it is not ready to do its real “work”. For example, it may still be loading libraries, waiting for a database connection, building some internal caches, whatever…
* `ready` means that the container is running, fully functional and ready to process incoming traffic.

By extension, we can define POD states as:
* `starting` if a POD has at least one container in the `starting` state;
* `ready` if all the containers of a POD are `ready`.

In today’s Kubernetes terminology, we don’t distinguish those states and both are named `running`.

### Need for throttling

The initialization of the containers might be expensive in terms of CPU or I/O.
We have examples of applications that:
* load a huge number of dynamic libraries and the symbol relocation takes a significant amount of time;
* retrieve some configuration from a file or a remote DB and denormalize that configuration, this can be expensive as well;
* create some local caches that need to be fed before the application can work;
* etc.

A starting container which consumes a lot of resources can have two kind of consequences:

1. It slows down the `ready` containers which are already running on the same machine and cause a response time degradation of the services provided by PODs that are running on the same machine than the starting one.
2. It slows down other `starting` containers which take longer to become `ready`.

#### 1. concurrency between the `starting` POD and the `ready` ones which are running on the same machine

This issue can be solved by adjusting the priorities of the different PODs so that one POD cannot starve others.

#### 2. concurrency between the `starting` containers and the other `starting` ones of the same POD

This increases the `starting` time of the processes themselves (resource starvation).

Our processes have internal health checks that check that the `starting` phase does not last longer than a pre-defined time-out.

In case of resource starvation, the time-out expires and the startup is considered failed.

We previously solved this by limiting the number of processes that can start simultaneously in order to avoid resource starvation and have more predictable start-up times.

## Protect the `ready` PODs from the `starting` ones on the same machine

Let’s consider the situation where, on a given machine, we have:
* a POD in the `ready` state which is not consuming a lot of CPU, but which is handling requests and their response times are critical.
* a POD in the `starting` state which is very CPU greedy.

We want to prevent the `starting` POD from starving the `ready` one in order to protect the response times of the processes of the `ready` POD. We must guarantee that even if the `starting` POD has many more containers than the `ready` one.

Today, docker containers are spawned in their own cgroups, all children of `/system.slice`. As a consequence, all the containers have the same weight. If the `starting` POD has ten times more containers than the `ready` one, it will be allocated ten times more CPU.

Having an additional layer with a slice per POD would allow to have a fair resource allocation per POD instead of having a fair resource allocation per container.

We propose to enhance docker to be able to specify the parent slice of containers’ cgroup ([Pull request #9436](https://github.com/docker/docker/pull/9436), [Pull request #9551](https://github.com/docker/docker/pull/9551)) and to enhance kubernetes to create one slice per POD.

Before:
```
systemd-cgls
├─1 /sbin/init
├─system.slice
│ ├─docker-123456….scope
│ │ └─100 /foo/bar/baz
│ ├─docker-123457….scope
│ │ └─101 /foo/bar/baz
│ ├─docker-123458….scope
│ │ └─103 /foo/bar/baz
│ ├─docker-123459….scope
│ │ └─104 /foo/bar/baz
```

After:
```
systemd-cgls
├─1 /sbin/init
├─system.slice
│ ├─kubernetes.slice
│ │ ├─k8s_pod_X.slice
│ │ │ ├─docker-123456….scope
│ │ │ │ └─100 /foo/bar/baz
│ │ │ └─docker-123457….scope
│ │ │   └─101 /foo/bar/baz
│ │ └─k8s_pod_Y.slice
│ │   ├─docker-123458….scope
│ │   │ └─103 /foo/bar/baz
│ │   └─docker-123459….scope
│ │     └─104 /foo/bar/baz
```

## Limit the number of processes that are starting at a given time

To avoid overloading a machine, rather than start a container as soon as its image has been pulled, we propose to control that start with a policy.

When a POD is assigned to a minion, kubelet creates the following FSM for each container:

![container FSM](https://rawgithub.com/lhuard1A/throttling-study/master/throttling_fsm.svg)

When an image is pulled, the container is not started immediately. Instead, it enters a `pending` state.

The `pending` to `starting` transition is triggered when the throttling policy authorizes it. The throttling policy is described below.

The `starting` to `ready` transition is triggered when the container is considered ready.

### Readiness detection

The `starting` to `ready` transition is new. This transition can be triggered in two ways:
* either the containers send a notification to say they are ready;
* or the containers are polled by docker/kube to check their readiness.

#### `ready` notification

This solution requires the containers to notify their readiness.

systemd has a similar requirement and lets services notify their readiness via different means:
* simple: the service is immediately ready;
* forking: the service is ready as soon as the parent process exits;
* dbus: the service is ready as soon as it acquires a name on D-Bus
* notify: the service is ready as soon at it has explicitly notified systemd about it by posting a message on a dedicated UNIX socket via the `sd_notify` function.

For containers, the `notify` service type sounds to be the most suitable.

##### Pros

* Notification is sent as soon as possible.

##### Cons

* Either it introduces some “systemd” dependency, or it requires to implement another notification mechanism inspired by the `sd_notify` feature. In all cases, it is intrusive since it requires to implement something in the containerized processes.
* This may be perceived as an advantage because existing programs may already implement this `sd_notify` call.
  In practice, when possible, it is preferable to decouple the public resource allocation (socket binding for example) from the program start-up. Concretely, the sockets are bound by systemd itself, the program is `Type=simple` (considered as ready immediately) and the program receives the file descriptor of the socket.

#### `ready` polling

This solution consists in having docker/kube regularly check the `readiness` of containers.

Such a mechanism already exists in Kubernetes as LivenessProbe. There are already different flavours of LivenessProbes:
* HTTP probe: try to do an HTTP GET on a given URL
* TCP probe: try to connect on a given port
* Exec: try to execute an arbitrary command inside the container

The last one sounds generic enough to implement anything.

##### Pros

* LivenessProbe is a mechanism that already exists;
* It is not intrusive since it doesn’t require to implement an `sd-notify`-like call in the programs.

##### Cons

* It is based on “polling”.
* More fork/exec.

#### Preferred solution

Reusing the LivenessProbe mechanism.

### Throttling policy

This is what triggers the `pending` to `starting` transition.

Several strategies are possible:

#### limit the number of processes in the `starting` state

This policy allows a container to go from `pending` to `starting` as soon as the number of containers in `starting` state on the minion (whatever the POD it belongs to) drops below a pre-defined threshold.

##### Pros

* Simple

##### Cons

* It’s non-trivial to set the threshold.

  If the processes are CPU bound and are consuming 100% of a CPU when they are in `starting` state, then, the optimal threshold would be the number of cores of the machine.

  If the processes are mostly waiting for external resources, then, the above recommendation is suboptimal.

  If the processes are multi-threaded and are consuming 100% of 3 CPUs, then, the optimal threshold is the third of the number of cores of the machine.

* If processes are stuck in the `starting` state, they will prevent other containers from being started. It is thus mandatory to implement a time-out mechanism that passes the `starting` containers in `failed` state if they stay in `starting` for too long.

#### time throttling

This policy allows only a maximum number of containers to become `starting` per a given amount of time. Ex: 2 containers per minion can be started per second.

##### Pros

* Simple
* Ensures that all containers startup will be _triggered_ within a predictible time, giving a chance to all containers to become ready.

##### Cons

* Does not guarantee that:
  * The machine is never overloaded
  * The resources of the machines are optimized (we don’t uselessly throttle a container whereas the CPU is idle, there is no I/O, etc.)

#### resource monitoring

This policy allows a container to become `starting` only if:
* the CPU consumption drops below a given threshold during a given amount of time
* the I/O usage drops below a given threshold during a given amount of time
* the load average of the machine drops below a given threshold

##### Pros

* Really takes into account the available resource of the machine and its usage to optimize usage without overloading it

##### Cons

* Implementation is more complex
* If the machine is loaded by things other than `starting` containers (like `ready` containers or even processes running on the machine that are not docker containers), it will prevent containers from starting.

#### Composite policy

The “resource monitoring” policy is the one that better uses the resources but it relies on resource consumption averaged for a given amount of time.

It should be combined with a maximum number of `starting` containers and a maximum start-up rate in order to not start too many containers before the “average CPU for last 10s” or “average I/O for last 10s” or “load average for 1min” increases.

If we limit the maximum number of `starting` containers, we must have a time-out mechanism that prevents containers from staying in `starting` for too long.

If the resource consumption doesn’t drop when containers leave the `starting` state — either because the `ready` containers also consume resources, or because some resources are consumed by processes outside Kubernetes/OpenShift — it will prevent `pending` containers from starting for ever. In order to avoid that, we need to have a minimum start-up rate that guarantees that we will eventually start all the containers.

##### Pros

* The only one that works in all cases?

##### Cons

* Complex

### Configuration example

The throttling mechanism described in this section is about avoiding resource starvation. The resources are global to the machine. As a consequence, the settings cannot be at the the POD level. They need to be at the minion level

We could have a configuration file attached to minions.

`m1_config.json`:
```json
{
  "kind": "MinionConfig",
  "apiVersion": "v1beta1",
  "throttling": {
    "maxStartingContainersPerCore": "3",
    "maxLoadAverageMulitplier": "1.5",
    "maxCPU": "80%",
    "minRate": "0.1",
    "maxRate": "10"
  }
}
```

```sh
kubecfg -c m1_config.json update minions/192.168.10.1
```

* **maxStartingContainersPerCore**: The maximum number of containers per core that could be in the `starting` state on the machine.
* **maxLoadAverageMultiplier**: Can be a factor multiplied by the number of cores of the machine
* **minRate**: Whatever the other settings, we’ll start at least one container every 10s (i.e: 0.1 container per second)
* **maxRate**: Whatever the other settings, we’ll start at most 10 containers per second (i.e: 1 container every 100ms)

## Dependency management proposal

“A POD is a collocated group of containers […] which are tightly coupled — in a pre-container world, they would have executed on the same physical or virtual host.” (extract from the [PODs definition](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/pods.md))

As they are tightly coupled, there might be dependencies between containers. In a “pre-container world”, they would have been spawned on a host by an init system able to handle dependencies.

We propose to enhance Kubernetes to use a dependency graph inside PODs to decide which containers can be started and which containers must wait for others.

Let’s consider, as an example, a POD with 5 containers linked together by the following dependencies:

![dependency graph](https://rawgithub.com/lhuard1A/throttling-study/master/dependency_graph.svg)

Such a dependency graph could be described in the POD json like this:
```json
{
  "kind": "Pod",
  "apiVersion": "v1beta1",
  "id": "app",
  "desiredState": {
    "manifest": {
      "version": "v1beta1",
      "id": "app",
      "containers": [{
        "name": "app_a",
        "image": "me/app_a",
        "livenessProbe": {
          "exec": {
            "command": "/check_A_health"
          }
        }
      },{
        "name": "app_b",
        "image": "me/app_b",
        "livenessProbe": {
          "exec": {
            "command": "/check_B_health"
          }
        },
        "dependsOn": [
          "mag"
        ]
      },{
        "name": "app_c",
        "image": "me/app_c",
        "livenessProbe": {
          "exec": {
            "command": "/check_C_health"
          }
        },
        "dependsOn": [
          "app_b"
        ]
      },{
        "name": "app_d",
        "image": "me/app_d",
        "livenessProbe": {
          "exec": {
            "command": "/check_D_health"
          }
        },
        "dependsOn": [
          "app_b"
        ]
      },{
        "name": "app_e",
        "image": "me/app_e",
        "livenessProbe": {
          "exec": {
            "command": "/check_E_health"
          }
        },
        "dependsOn": [
          "app_c",
          "app_d"
        ]
      }]
    }
  },
  "labels": {
    "name": "app"
  }
}
```

The POD start-up sequence is amended so that, when a POD is assigned to a minion, kubelet creates the following FSM for each container:

![container FSM](https://rawgithub.com/lhuard1A/throttling-study/master/dependency_fsm.svg)

The containers are initially in the `downloading` state. Once the image is pulled, they move to the new `blocked` state.

When a container X reaches the `blocked` state, the state of all its dependencies are checked. If all of them are `ready`, the container X moves to `starting` immediately. This is for example always the case for the containers which have no dependency.

Then, when a container X passes the `starting` to `ready` transition, for each container Yi in the `blocked` state that depends on X, we check the state of all the dependencies of Yi. If all of them are `ready`, then the Yi container becomes `starting`.

In the example above, when the `fe` container becomes `ready`, the state of `cs` is checked. It it’s `ready`, then the state of `example` moves from `blocked` to `starting`.

### Cycle detection

We must ensure that the dependency graph is a Directed Acyclic Graph (DAG), that is to say that we do not have circular dependencies among our containers.

This could, for example, be checked when parsing the json if we enforce a rule saying that the `dependsOn` list of a container can only contain containers previously declared in the file.

## Combining throttling with dependency management

### Example of why dependency management needs to be combined with throttling

Let’s imagine a POD with 3 containers A, B and C. Those 3 containers can be started in any order in the sense that they won’t fail, crash or prematurely exit if the others are not there.

However, B and C needs to communicate with A in order to become `ready`.

For example:
* A is a database. It notifies its readiness as soon as it is ready to process requests.
* B is an application that needs to connect to the database to configure itself. It notifies its readiness as soon as it is configured.
* C is similar to B

The throttling settings limit the number of containers authorized to be in `starting` at the same time to 2.

We have no dependency expressed in the POD json.

If we are lucky, things can happen in this order:

* B starts. It cannot configure itself because A is not there. It is waiting for A.
* A starts.
* A is ready.
* As A becomes ready, there is one “starting” slot available to make C start.
* B connects to A. B configure itself and eventually becomes ready
* C connects to A, configure itself and becomes ready

![Lucky flow](https://rawgithub.com/lhuard1A/throttling-study/master/lucky_flow.svg)

Note that even if this works, before A becomes `ready`, B “consumes” a `starting` slot although it is not consuming resources. It is sub-optimal as, if we knew that B depends on A, we could have started a container of an other POD which doesn’t depend on A.

If we are unlucky, things can happen in this order:

* B and C are started. They are waiting for being able to connect to A
* A is not started because we already have two processes in the starting mode which is the maximum allowed by our policy.
* We’re dead-locked.

![Unlucky flow](https://rawgithub.com/lhuard1A/throttling-study/master/unlucky_flow.svg)

This trivial example shows that
* limiting the number of processes starting at a time and
* having process readiness conditioned by other process readiness

is not possible if we cannot enforce a start-up order. That’s why if the readiness of some containers depends on the readiness of others, this dependency needs to be handled as described in the previous section.

### Final picture

Here is the complete FSM of containers:

![Unluck flow](https://rawgithub.com/lhuard1A/throttling-study/master/dependency_throttling_fsm.svg)

Some states have associated actions that are triggered when the state is entered:
* `downloading`: the image is fetched with a `docker pull`;
* `blocked`: the container can be created with `docker create` but not started;
* `starting`: the container is started with `docker start`.

The transitions are defined as followed:
* When a POD is assigned to a minion, all its containers are put in the `downloading` state.
* When the image is pulled, the container becomes `blocked`.
* Immediately, the dependencies of that container are checked. If they are all in the `ready` state, then the container becomes `pending` immediately, otherwise, it stays in `blocked` state.
* When there are containers in the `pending` state, the throttling mechanism regularly checks if conditions are met to take a `pending` container and make it progress to `starting`.
* LivenessProbes regularly check if `starting` containers are `ready`. The first time a LivenessProbe reports a success, the container becomes `ready`.
* If a container remains `starting` for longer time than a pre-defined threshold without its LivenessProbe reporting success, the container becomes `failed`.
* Each time a container leaves the `starting` state, we check all the dependencies of the `blocked` containers that depend on it. If they are all `ready`, then that `blocked` container becomes `pending`.
