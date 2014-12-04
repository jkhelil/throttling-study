# Throttling study

## Current start-up flow

The current flow when a POD is started by kubernetes is composed of the following steps:

1. `scheduler` binds the POD to a minion. This leaves the state of the POD in `Pending` status.
2. `kubelet` pulls the images of the PODs. This leaves the state of the POD in `Pending` status.
3. When a docker images is pulled, the container is started immediately.
4. Once all the containers are running, `kubelet` passes the status of the POD to `Running`.

![Current state machine](https://rawgithub.com/lhuard1A/throttling-study/master/current_state_machine.svg)

## Running does not imply Ready

The first remark is that it is not because a process is started that it is ready to work and process traffic.

For example, we can have a process that starts its life by fetching some configuration elements from somewhere and doing some expensive configuration stuff before being really ready to work.

Further in this document, we’ll make a distinction between `starting` and `ready` states.
* `starting` means that the container is running, the process is live, but it is not ready to do its real “work”. For example, it may still be loading libraries, waiting for a database connection, building some internal caches, whatever…
* `ready` means that the container is running, is fully functional and is ready to process incoming traffic.

In today’s Kubernetes terminology, we don’t distinguish those two states and both are named `running`.

## Need for throttling

### Throttling the starting containers

The initialization of the containers might be CPU expensive.
We have examples of applications that:
* load a huge number of dynamic libraries and the symbol relocation takes a significant amount of time;
* retrieve some configuration from a file or a remote DB and denormalize that configuration, this can be expansive as well;
* create some local caches that need to be fed before the application can work;
* etc.

A starting container which consumes a lot of resources can have two kind of consequences:

1. It slows down the `ready` containers which are already running on the same machine and cause a response time degradation of the services provided by PODs that are running on the same machine than the starting one.
2. It slows down the `starting` containers themselves which are longer to become `ready`.

#### 1. concurrency between the `starting` POD and the `ready` ones which are running on the same machine

This issue can be solved by adjusting the priorities of the different PODs so that one POD cannot starve other ones.

#### 2. concurrency between the `starting` containers and the other `starting` ones of the same POD

The impact is that it slows down the `starting` time of the processes themselves because of resource starvation.

The main issue we have is that our processes have internal health checks that check that the `starting` phase does not last longer than a pre-defined time-out.

In case of resource starvation, the times-out exhaust and the programs think they fell in an infinite loop or something.

We use to solve that issue that limiting the number of processes that can start simultaneously in order to avoid resource starvation and to have more predictable start-up times.

### Throttling the images pull

If too many machines are pulling too many images from the registry at the same time, it may hurt the docker registry.

In order to keep it reasonably loaded, we may want to limit the number of images that can be pulled at the same time.

An other solution would be to rely on network QoS.

Anyhow, the throttling of images pull is not in the scope of this paper.

## Throttling proposition

### Limiting the number of processes `starting` at a given time

This proposition consists in controlling how many containers can be in the `starting` state at a time. When that pre-defined number of `starting` container is reached, the other containers that are ready to be started stay in a `pending` state. When a `starting` container becomes `ready`, a one in `pending` can become `starting`.

The startup sequence of a POD is modified to become:

![Proposed state machine](https://rawgithub.com/lhuard1A/throttling-study/master/proposed_state_machine.svg)

#### Requirements

Implementing this solution requires to be able to distinguish `starting` from `ready`, so we need a way to know when a container is ready.

#### Limitation

Let’s assume the following hypothesis:

* We have processes able to notify kube that they are ready

* Inside a POD, we have:
  * a container A
  * a container B that needs to communicate with A in order to become ready
  * a container C that needs to communicate with A in order to become ready

  An example could be:
  * A is a database. It notifies its readiness as soon as it is ready to process requests.
  * B is an application that needs to connect to the database to configure itself. It notifies its readiness as soon as it is configured.
  * C is similar to B

* We set the throttling parameter to limit the number of processes starting at a given time to 2.

If we are lucky, things can happen in this order:

* B starts. It cannot configure itself because A is not there. It is waiting for A.
* A starts.
* A is ready.
* As A becomes ready, there is one “starting” slot available to make C start.
* B connects to A. B configure itself and eventually becomes ready
* C connects to A, configure itself and becomes ready

![Lucky flow](https://rawgithub.com/lhuard1A/throttling-study/master/lucky_flow.svg)

If we are unlucky, things can happen in this order:

* B and C are started. They are waiting for being able to connect to A
* A is not started because we already have two processes in the starting mode.
* We’re dead-locked.

![Unluck flow](https://rawgithub.com/lhuard1A/throttling-study/master/unlucky_flow.svg)

This trivial example shows that
* limiting the number of processes starting at a time and
* having process readiness conditioned by other process readiness

is not possible if we cannot enforce a start-up order.

Having to specify a start-up order and a maximum number of starting containers sounds like being micro management which is both complex to implement and to use.

Moreover, how should the maximum number of starting containers be chosen?

* If the starting containers are CPU bound, limiting the number of containers starting simultaneously to the number of cores will optimize the resource usage.
* If the starting containers are mostly waiting for external resources, setting the maximum number of containers starting simultaneously in order to optimize the resource usage is much trickier.

### Fine control the resource allocation with cgroup settings

This proposition consists in letting the containers start as soon as they are ready to start, as soon as the corresponding image has been pulled.

In order to avoid `starting` containers to starve the other ones, the CPUShares and BlockIOWeight cgroup parameters can be adjusted so that the containers in `ready` PODs have more priority than containers in `starting` PODs.

#### Requirements

This proposition also requires to be able to distinguish `starting` from `ready`.

#### Limitation

This solution consists in letting all the containers start as soon as their images are pulled. But, the resources allocated to `starting` containers is limited in order to not starve the other processes running on the box.

The consequence is that the time needed for a process to become `ready` may be long and very dependent on the hosting machine load.

If those processes have a watchdog that limits `start-up` time, the corresponding time-out may need to be increased.

## `starting` to `ready` transition

Both throttling propositions described above require to be able to distinguish `starting` state from `ready` one.

This transition can be triggered by two ways:
* either the containers send a notification to tell they are ready;
* or the containers are polled by docker/kube to check their readiness.

### `ready` notification

This solution requires the containers to notify their readiness.

systemd has a similar requirement and let the systemd services notify their readiness via different means:
* simple: the service is immediately ready;
* forking: the service is ready as soon as the parent process exits;
* dbus: the service is ready as soon as it acquires a name on D-Bus
* notify: the service is ready as soon at it has explicitly notified systemd about it by posting a message on a dedicated UNIX socket via the `sd_notify` function.

For containers, the `notify` service type sounds to be the most suitable.

#### Pros

* Notification is sent as soon as possible.

#### Cons

* Introduce some "systemd" dependency.
* This may be perceived as an advantage because existing programs may already implement this `sd_notify` call.
  In practice, when possible, it is preferable to decouple the public resource allocation (socket binding for example) from the program start-up. Concretely, the sockets are bound by systemd itself, the program is `Type=simle` (considered as ready immediately) and the program receives the file descriptor of the socket.

### `ready` polling

This solution consists in having docker/kube regularly check the `readiness` of containers.

Such a mechanism already exists in Kubernetes as LivenessProbe. There is already different flavours of LivenessProbes:
* HTTP probe
* TCP probe
* Run a command inside the container

The last one sounds generic enough to implement anything.

#### Pros

* LivenessProbes already exist in Kubernetes and the “run a command” one is very generic.

#### Cons

* Based on polling

## Proposition

Regarding the previously described propositions, their pros and cons, I propose for the initial implementation of the start-up throttling:

1. All the containers are spawned in their own cgroup. Instead of spawning all of them in the `/system.slice` slice, the containers cgroup are created in a slice dedicated to the POD:

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

2. The POD json does not contain `CPUShares`, `BlockIOWeight` and other cgroup settings at container level, but also at POD level.

3. The `CPUShares`, `BlockIOWeight` and other cgroup settings are supplemented with `StartupCPUShares`, `StartupBlockIOWeight`, etc.

4. The current `running` state of PODs is split between `starting` and `ready`.
  * If there are containers the image of which is still being pulled, the POD status is `pending`;
  * If all the containers are started but the livenessProbe of some of them is still not OK, the POD status is `starting`
  * If all the containers are started and the livenessProbe of all of them is OK, the POD status is `ready`

5. When a POD is created (kubelet notices that the POD has been bound to its minion),
  * A new cgroup slice is created: `/system.slice/kubernetes.slice/k8s_pod_x.slice` with the `StartupCPUShares` and `StartupBlockIOWeight` values.
  * The containers are spawned inside that slice.
  * When the POD reaches the `ready` state, the cgroup settings are set to the `CPUShares` and `BlockIOWeight` values.

### Hi level gaps

* Add in docker, the ability to specify a slice.
* Make kubernetes use the LivenessProbes of containers to change the “state” of the POD.
* Add in kubernetes, the ability to specify `[Startup]CPUShares` and `[Startup]BlockIOWeight` at POD level in addition to container level.
* Make kubernetes update the `cpu.shares` and `blkio.weight` of the POD slice when the state of POD switches to `ready`.
