# Nestybox: Sysbox-Pods Feature Preview

***

**NOTE: This is a temporary repo to allow users to preview the upcoming "Sysbox pods"
feature. The feature is only present via this repo. It's not yet in the [Sysbox](https://github.com/nestybox/sysbox) or
[Sysbox Enterprise](https://github.com/nestybox/sysbox-ee) repos, but it's
expected to land there in a few weeks (ideally after we've received
sufficient feedback).**

**Once the feature is released, this current repo will cease to exist and the
documentation and other artifacts will be moved to the Sysbox repo.**

**Having said this, we would appreciate if you refer to Sysbox repo for any GitHub
star that you may be willing to throw at us -- always welcomed!**

***

The Sysbox pods feature enables deployment of Kubernetes (K8s) pods using the
[Sysbox](https://github.com/nestybox/sysbox) runtime.

In essence, Sysbox pods value-proposition is twofold: extended functionality
and enhanced security.

With Sysbox, K8s can deploy strongly isolated (rootless) pods that can run not
just microservices, but also workloads such as systemd, Docker, Podman (WIP)
and even K8s, seamlessly.

Prior to Sysbox, running such pods required using privileged (insecure) containers
and very complex pod setups and entrypoints. This is insecure (privileged pods
allow users inside the pod to easily compromise the K8s host) and puts a lot of
complexity on the K8s cluster admin & users.

You can now use Sysbox for improving the security of your K8s pods by replacing
your existing privileged pods, and for deploying VM-like environments inside pods
(quickly and efficiently, without actually using virtual machines).

With Sysbox this insecurity and complexity go away: the pods are strongly
isolated, and Sysbox absorbs all the complexity of setting up the pod correctly
to run the above-mentioned workloads.

## Demo Video

Here is a video showing how to use K8s + Sysbox to deploy a rootless pod that runs
systemd and Docker inside:

https://asciinema.org/a/401488?speed=1.5

## Contents

*   [Installation](#installation)
*   [Kubernetes Version Requirements](#kubernetes-version-requirements)
*   [Kubernetes Node Requirements](#kubernetes-node-requirements)
*   [CRI-O Installation](#cri-o-installation)
*   [Sysbox Installation](#sysbox-installation)
*   [Sysbox Pods Deployment](#sysbox-pods-deployment)
*   [Kubernetes Manifests](#kubernetes-manifests)
*   [Sysbox Container Images](#sysbox-container-images)
*   [Host Volume Mounts](#host-volume-mounts)
*   [Uninstallation](#uninstallation)
*   [Troubleshooting](#troubleshooting)
*   [Contact](#contact)
*   [Thank You](#thank-you)

## Installation

Sysbox can be installed in all or some of the Kubernetes cluster worker nodes,
according to your needs.

Installation is done via a daemonset which "drops" the Sysbox binaries onto the
desired K8s nodes (steps are shown later).

Installing Sysbox on a node does not imply all pods on the node are deployed
with Sysbox. You can choose which pods use Sysbox via the pod's spec. Pods that
don't use Sysbox continue to use the default low-level runtime (i.e., the OCI
runc) or any other runtime you choose.

Pods deployed with Sysbox are managed via K8s just like any other pods; they can
live side-by-side with non Sysbox pods and can communicate with them according
to your K8s networking policy.

## Kubernetes Version Requirements

Sysbox is only supported on Kubernetes v1.20.\* at this time.

The reason for this is that Sysbox currently requires the presence of the CRI-O
runtime v1.20, as the latter introduces support for [rootless pods](docs/k8s-setup.md#installing-cri-o-on-a-k8s-worker-node).
Since the version of CRI-O and K8s must match, the K8s version must also be v1.20.\*.

## Kubernetes Node Requirements

Once you have a K8s v1.20 cluster up and running, you need to install Sysbox
on one or more worker nodes.

Prior to installing Sysbox, ensure each node where you will install Sysbox meets
the following requirements:

1.  The node's OS must be Ubuntu Focal or Bionic (with a 5.0+ kernel).

2.  The shiftfs kernel module should be present (if you want to mount host files
    or directories into sysbox-based pods).

3.  [rsync](https://packages.ubuntu.com/search?keywords=rsync) must be installed
    on the worker node.

4.  CRI-O v1.20 must be installed and running on the node.

## CRI-O Installation

**NOTE: If the Kubernetes worker nodes where you want to install Sysbox already
have CRI-O v1.20, you can skip this step. However you should double-check that
CRI-O is configured as described [here](docs/k8s-setup.md#cri-o-configuration).**

Nestybox has created a K8s daemonset that installs CRI-O on the desired Kubernetes
worker nodes. The daemonset "drops" CRI-O on the worker nodes and configures the
K8s Kubelet to use CRI-O. In essence, CRI-O replaces the existing CRI on the
nodes (e.g., dockershim or containerd).

To deploy the CRI-O installation daemonset do the following:

1.  Add the K8s label "crio-install=yes" to the worker nodes where CRI-O
    should be installed.

```console
kubectl label nodes <node-name> crio-install=yes
```

You should only label K8s worker nodes. Do NOT label K8s master nodes because
the CRI-O installation daemonset will restart the Kubelet, thus bringing down the node
temporarily (and you don't want to bring down the K8s control-plane in the
process).

2.  Deploy the CRI-O installation daemonset:

```console
kubectl apply -f https://raw.githubusercontent.com/nestybox/sysbox-pods-preview/master/k8s-manifests/rbac/crio-deploy-rbac.yaml
```

```console
kubectl apply -f https://raw.githubusercontent.com/nestybox/sysbox-pods-preview/master/k8s-manifests/daemonset/crio-deploy-k8s.yaml
```

This will cause K8s to run the CRI-O installation daemonset on all nodes
labeled with `crio-install=yes` in the prior step. The daemonset will
"drop" CRI-O into the node and restart the Kubelet. The process can
take several seconds (e.g., 30 secs). After this, the daemonset will
remain idle until deleted.

**NOTE: Do not delete the daemonset unless you want to remove CRI-O from the
worker node(s).**

3.  Verify all is good:

*   You should see the "crio-deploy-k8s" daemonset running.

*   The installation daemonset will add a label to the node:
    `crio-runtime=running`.

*   Each node will have a "crio-deploy-k8s" pod running. The pods logs
    should look like this:

```console
$ kubectl -n kube-system logs -f pod/crio-deploy-k8s-d4ckz
Deploying CRI-O installer agent on the host ...
Running CRI-O installer agent on the host (may take several seconds) ...
Stopping the CRI-O installer agent on the host ...
Removing CRI-O installer agent from the host ...
Configuring CRI-O ...
Restarting CRI-O ...
Restarting Kubelet ...
```

*   In each worker node where the daemonset was deployed, you should see
    CRI-O running properly, and kubelet running properly.

```console
systemctl status crio
systemctl status kubelet
```

Once CRI-O is installed, the next step is to install Sysbox.

## Sysbox Installation

Assuming the K8s worker nodes meet the above requirements, installing Sysbox is
easily done via a daemonset as follows:

1.  Add the K8s label "sysbox-install=yes" to the nodes where Sysbox will be
    installed.

```console
kubectl label nodes <node-name> sysbox-install=yes
```

You should only label K8s worker nodes. Do NOT label K8s master nodes because
the Sysbox installation daemonset will restart CRI-O, thus bringing down the
node temporarily (and you don't want to bring down the K8s control-plane in the
process).

2.  Deploy the Sysbox installation daemonset:

```console
kubectl apply -f https://raw.githubusercontent.com/nestybox/sysbox-pods-preview/master/k8s-manifests/rbac/sysbox-rbac.yaml
```

```console
kubectl apply -f https://raw.githubusercontent.com/nestybox/sysbox-pods-preview/master/k8s-manifests/daemonset/sysbox-deploy-k8s.yaml
```

This will cause K8s to run the sysbox installation daemonset on all nodes
labeled with `sysbox-install=yes` in the prior step. The daemonset will
"drop" Sysbox into the node and restart CRI-O. After this the daemonset will
remain idle until deleted.

**NOTE: The CRI-O restart is necessary in order for it to pick up the presence of
the Sysbox runtime. Unfortunately this will temporarily disrupt all pods on the
nodes where Sysbox is installed, for ~1 minute. For example the output of
`kubectl get all --all-namespaces` will show errors on pods deployed on the
affected nodes. After about 1 minute, those pods should return to "running"
state. See the [troubleshooting doc](docs/troubleshoot.md) for more info.**

3.  Verify all is good:

*   You should see the "sysbox-deploy-k8s" daemonset running.

*   The installation daemonset will add a label to the node:
    `sysbox-runtime=running`. This label means sysbox is running on the node.

*   Each node will have a "sysbox-deploy-k8s" pod running.

*   In each worker node where the daemonset was deployed, you should see
    Sysbox and CRI-O running properly.

```console
systemctl status sysbox
systemctl status crio
```

4.  Add the Sysbox "runtime class" resource to K8s.

```console
kubectl apply -f https://raw.githubusercontent.com/nestybox/sysbox-pods-preview/master/k8s-manifests/runtime-class/sysbox-runtimeclass.yaml
```

The runtime class informs Kubernetes that there is a new container runtime
called "sysbox-runc" and that it's present on nodes labeled with
"sysbox-runtime=running".

That's it for installation! Next comes the fun part, deploying the rootless pods with Sysbox ...

## Sysbox Pods Deployment

Deploying pods with Sysbox is easy: you only need a couple of
things in the pod spec.

For example, here is a sample pod spec using the
`ubuntu-bionic-systemd-docker` image. It creates a rootless pod that
runs systemd as init (pid 1) and comes with Docker (daemon + CLI) inside:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubu-bio-systemd-docker
  annotations:
    io.kubernetes.cri-o.userns-mode: "auto:size=65536"
spec:
  runtimeClassName: sysbox-runc
  containers:
  - name: ubu-bio-systemd-docker
    image: registry.nestybox.com/nestybox/ubuntu-bionic-systemd-docker
    command: ["/sbin/init"]
  restartPolicy: Never
```

There are two key pieces of the pod's spec that tie it to Sysbox:

*   "runtimeClassName": Tells K8s to deploy the pod with Sysbox (rather than the
    default OCI runc). The pods will be scheduled only on the nodes that support
    Sysbox.

*   "io.kubernetes.cri-o.userns-mode": Tells CRI-O to launch this as a rootless
    pod (i.e., root user in the pod maps to an unprivileged user on the host)
    and to allocate a range of 65536 Linux user-namespace user and group
    IDs. This is required for Sysbox pods.

Also, for Sysbox pods you typically want to avoid sharing the process (pid)
namespace between containers in a pod. Thus, avoid setting
`shareProcessNamespace: true` in the pod's spec, especially if the pod carries
systemd inside (as otherwise systemd won't be pid 1 in the pod and will fail).

Depending on the size of the pod's image, it may take several seconds for the
pod to deploy on the node. Once the image is downloaded on a node however,
deployment should be very quick (few seconds).

## Kubernetes Manifests

The K8s manifests used for setting up Sysbox can be found [here](k8s-manifests).

## Sysbox Container Images

The pod in the prior example uses the
`ubuntu-bionic-systemd-docker`, but you can use any container
image you want. Sysbox places no requirements on the container image.

Nestybox has several images which you can find here:

https://hub.docker.com/u/nestybox

Those same images are in the Nestybox GitHub registry (`ghcr.io/nestybox/<image-name>`).

We usually rely on `registry.nestybox.com` as an image front-end so that docker
image pulls are forwarded to the most suitable repository without impacting our
users.

Some of those images carry systemd only, others carry systemd + Docker, other
carry systemd + K8s (yes, you can run K8s inside rootless pods deployed by
Sysbox).

## Host Volume Mounts

To mount host volumes into a K8s pod deployed with Sysbox, the K8s worker node's
kernel must include the `shiftfs` kernel module (see
[here](docs/k8s-setup.md#installing-shiftfs-on-a-k8s-worker-node) for installation
info).

This is because such Sysbox pods are rootless, meaning that the root user inside
the pod maps to a non-root user on the host (e.g., pod user ID 0 maps to host
user ID 296608). Without shiftfs, host directories or files which are typically
owned by users IDs in the range 0->65535 will show up as `nobody:nogroup` inside
the pod.

The shiftfs module solves this problem, as it allows Sysbox to "shift" user
and group IDs inside the pod, such that files owned by users 0->65536 on the
host also show up as owned by users 0->65536 inside the pod.

Once shiftfs is installed, Sysbox will detect this and use it when necessary.
As a user you don't need to know anything about shiftfs; you just setup the pod
with volumes as usual.

For example, the following spec creates a Sysbox pod with ubuntu-bionic + systemd +
docker and mounts host directory `/root/somedir` into the pod's `/mnt/host-dir`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubu-bio-systemd-docker
  annotations:
    io.kubernetes.cri-o.userns-mode: "auto:size=65536"
spec:
  runtimeClassName: sysbox-runc
  containers:
  - name: ubu-bio-systemd-docker
    image: registry.nestybox.com/nestybox/ubuntu-bionic-systemd-docker
    command: ["/sbin/init"]
    volumeMounts:
      - mountPath: /mnt/host-dir
        name: host-vol
  restartPolicy: Never
  volumes:
  - name: host-vol
    hostPath:
      path: /root/somedir
      type: Directory
```

When this pod is deployed, Sysbox will automatically enable shiftfs on the pod's
`/mnt/host-dir`. As a result that directory will show up with proper user-ID and
group-ID ownership inside the pod.

With shiftfs you can even share the same host directory across pods, even if
the pods each get exclusive Linux user-namespace user-ID and group-ID mappings.
Each pod will see the files with proper ownership inside the pod (e.g., owned
by users 0->65536) inside the pod.

## Uninstallation

To uninstall Sysbox, see [here](docs/k8s-setup.md#sysbox-uninstallation).

To uninstall CRI-O, see [here](docs/k8s-setup.md#cri-o-uninstallation).

## Troubleshooting

Please contact us if you hit any problems (see contact info below).

The [troubleshooting](docs/troubleshoot.md) doc has some useful info too.

## Contact

Slack: [Nestybox Slack Workspace][slack]

Email: contact@nestybox.com

We are available from Monday-Friday, 9am-5pm Pacific Time.

## Thank You

We thank you **very much** for giving sysbox-pods an early try. We appreciate
any feedback you can provide to us!

[slack]: https://nestybox-support.slack.com/join/shared_invite/enQtOTA0NDQwMTkzMjg2LTAxNGJjYTU2ZmJkYTZjNDMwNmM4Y2YxNzZiZGJlZDM4OTc1NGUzZDFiNTM4NzM1ZTA2NDE3NzQ1ODg1YzhmNDQ#/
