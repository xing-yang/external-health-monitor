# External Health Monitor

The External Health Monitor is part of Kubernetes implementation of [Container Storage Interface (CSI)](https://github.com/container-storage-interface/spec). It was introduced as an Alpha feature in Kubernetes v1.19.

## Overview

The [External Health Monitor](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/1432-volume-health-monitor) is impelmeted as two components: `External Health Monitor Controller` and `External Health Monitor Agent`.

- External Health Monitor Controller:
  - The external health monitor controller will be deployed as a sidecar together with the CSI controller driver, similar to how the external-provisioner sidecar is deployed.
  - Trigger controller RPC to check the health condition of the CSI volumes.
  - The external controller sidecar will also watch for node failure events. This component can be enabled via a flag.

- External Health Monitor Agent:
  - The external health monitor agent will be deployed as a sidecar together with the CSI node driver on every Kubernetes worker node.
  - Trigger node RPC to check volume's mounting conditions.

The External Health Monitor needs to invoke the following CSI interfaces

- External Health Monitor Controller:
  - ListVolumes
  - ControllerGetVolume
- External Health Monitor Agent:
  - NodeGetVolumeStats

## Compatibility

This information reflects the head of this branch.

| Compatible with CSI Version | Container Image |
| ------------------------------------------------------------------------------------------ | -------------------------------|
| [CSI Spec v1.3.0](https://github.com/container-storage-interface/spec/releases/tag/v1.3.0) | k8s.gcr.io/sig-storage/csi-external-health-monitor-controller |
| [CSI Spec v1.3.0](https://github.com/container-storage-interface/spec/releases/tag/v1.3.0) | k8s.gcr.io/sig-storage/csi-external-health-monitor-agent |

## Driver Support

Currently the CSI volume health monitoring interfaces are only implemented in the Mock Driver.

## Usage

External Health Monitor needs to be deployed with CSI driver.

### Build && Push Image

You can run the command as below in the root directory of the project.

```bash
make container GOFLAGS_VENDOR=$( [ -d vendor ] && echo '-mod=vendor' )
```

And then, you can tag and push it to your own image repository.

```bash
docker tag csi-external-health-monitor-controller:latest <custom-image-repo-addr>/csi-external-health-monitor-controller:<custom-image-tag>

docker tag csi-external-health-monitor-agent:latest <custom-image-repo-addr>/csi-external-health-monitor-agent:<custom-image-tag>
```

### External Health Monitor Controller

```bash
cd external-health-monitor
kubectl create -f deploy/kubernetes/external-health-monitor-controller
```

### External Health Monitor Agent

```bash
kubectl create -f deploy/kubernetes/external-health-monitor-agent
```

You can run `kubectl get pods` command to confirm if they are deployed on your cluster successfully.

Check logs of external health monitor controller and agent as follows:

-  `kubectl logs <leader-of-external-health-monitor-controller-container-name> -c csi-external-health-monitor-controller`
-  `kubectl logs <external-health-monitor-agent-container-name> -c csi-external-health-monitor-agent`

Check if there are events on PVCs or Pods that report abnormal volume condition when the volume you are using is abnormal.

### CSI external health monitor controller sidecar command line options

#### Important optional arguments that are highly recommended to be used
* `--leader-election`: Enables leader election. This is useful when there are multiple replicas of the same external health monitor controller running for the same Kubernetes deployment. Only one of them may be active (=leader). A new leader will be re-elected when current leader dies or becomes unresponsive for ~15 seconds.

* `--leader-election-namespace <namespace>`: The namespace where the leader election resource exists. Defaults to the pod namespace if not set.

* `--metrics-address`: The TCP network address where the prometheus metrics endpoint will run (example: `:8080` which corresponds to port 8080 on local host). The default is empty string, which means metrics endpoint is disabled.

* `--metrics-path`: The HTTP path where prometheus metrics will be exposed. Default is `/metrics`.

* `--worker-threads`: Number of worker threads. Default value is 10.

#### Other recognized arguments
* `--monitor-interval <duration>`: Interval for the external health monitor controller to check volumes health condition. Default is 1 minute.

* `--kubeconfig <path>`: Path to Kubernetes client configuration that the external health monitor controller uses to connect to Kubernetes API server. When omitted, default token provided by Kubernetes will be used. This option is useful only when the snapshot controller does not run as a Kubernetes pod, e.g. for debugging.

* `--resync <duration>`: Internal resync interval of the external health monitor controller. Default is 10 minutes.

* `--csi-address`: Address of the CSI driver socket. Default is "/run/csi/socket".

* `timeout`:                  = flag.Duration("timeout", 15*time.Second, "Timeout for waiting for attaching or detaching the volume.")
        listVolumesInterval      = flag.Duration("list-volumes-interval", 5*time.Minute, "Time interval for calling ListVolumes RPC to check volumes' health condition")
        volumeListAndAddInterval = flag.Duration("volume-list-add-interval", 5*time.Minute, "Time interval for listing volumes and add them to queue")
        nodeListAndAddInterval   = flag.Duration("node-list-add-interval", 5*time.Minute, "Time interval for listing nodess and add them to queue")

* `--version`: Prints current external health monitor controller version and quits.

* All glog / klog arguments are supported, such as `-v <log level>` or `-alsologtostderr`.

### CSI external health monitor agent sidecar command line options

#### Important optional arguments that are highly recommended to be used
* `--csi-address <path to CSI socket>`: This is the path to the CSI driver socket inside the pod that the external-snapshotter container will use to issue CSI operations (`/run/csi/socket` is used by default).

* `--leader-election`: Enables leader election. This is useful when there are multiple replicas of the same external-snapshotter running for one CSI driver. Only one of them may be active (=leader). A new leader will be re-elected when current leader dies or becomes unresponsive for ~15 seconds.

* `--leader-election-namespace <namespace>`: The namespace where the leader election resource exists. Defaults to the pod namespace if not set.

* `--timeout <duration>`: Timeout of all calls to CSI driver. It should be set to value that accommodates majority of `CreateSnapshot`, `DeleteSnapshot`, and `ListSnapshots` calls. 1 minute is used by default.

* `snapshot-name-prefix`: Prefix to apply to the name of a created snapshot. Default is `snapshot`.

* `snapshot-name-uuid-length`: Length in characters for the generated uuid of a created snapshot. Defaults behavior is to NOT truncate.

* `--worker-threads`: Number of worker threads for running create snapshot and delete snapshot operations. Default value is 10.

#### Other recognized arguments
* `--kubeconfig <path>`: Path to Kubernetes client configuration that the CSI external-snapshotter uses to connect to Kubernetes API server. When omitted, default token provided by Kubernetes will be used. This option is useful only when the external-snapshotter does not run as a Kubernetes pod, e.g. for debugging.

* `--resync-period <duration>`: Internal resync interval when the CSI external-snapshotter re-evaluates all existing `VolumeSnapshotContent` instances and tries to fulfill them, i.e. update / delete corresponding snapshots. It does not affect re-tries of failed CSI calls! It should be used only when there is a bug in Kubernetes watch logic. Default is 60 seconds.

* `--version`: Prints current CSI external-snapshotter version and quits.

* All glog / klog arguments are supported, such as `-v <log level>` or `-alsologtostderr`.

## Community, discussion, contribution, and support

Learn how to engage with the Kubernetes community on the [community page](http://kubernetes.io/community/).

You can reach the maintainers of this project at:

- [Slack](https://kubernetes.slack.com/messages/sig-storage)
- [Mailing List](https://groups.google.com/forum/#!forum/kubernetes-sig-storage)

### Code of conduct

Participation in the Kubernetes community is governed by the [Kubernetes Code of Conduct](code-of-conduct.md).
