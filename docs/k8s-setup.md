# Kubernetes Cluster Preparation for Sysbox

This document describes how to prepare a Kubernetes cluster prior
to installing Sysbox.

These steps do not include the installation of Sysbox itself, which is
done easily via a daemonset as described in the [README file](../README.md#sysbox-installation).

## Contents

*   [K8s Master Node Setup](#k8s-master-node-setup)
*   [K8s Worker Node Setup](#k8s-worker-node-setup)
    *   [Installing Ubuntu Focal or Bionic](#installing-ubuntu-focal-or-bionic)
    *   [Installing CRI-O on a K8s Worker Node](#installing-cri-o-on-a-k8s-worker-node)
    *   [Configuring the K8s Kubelet to use CRI-O](#configuring-the-k8s-kubelet-to-use-cri-o)
    *   [Installing Shiftfs on a K8s Worker Node](#installing-shiftfs-on-a-k8s-worker-node)

## K8s Master Node Setup

Sysbox requires a Kubernetes cluster with version 1.20.\*.

The cluster should have at least two nodes, one acting as a master and one or
more worker nodes. Sysbox should be installed on the worker nodes only (i.e., we
don't recommend it be installed on the master nodes for reasons described later).

To setup such a cluster with `kubeadm`, you can run the following
command on the control plane node(s):

```console
$ sudo swapoff -a
$ sudo kubeadm init --kubernetes-version=v1.20.2 --pod-network-cidr=10.244.0.0/16
```

Since the K8s master node won't be running Sysbox pods, no further Sysbox related
preparations are required on it. The preparations are mainly on the K8s worker nodes
where Sysbox will be installed.

## K8s Worker Node Setup

The worker node setup is composed of 4 steps:

1.  Install Ubuntu Focal or Bionic (with a 5.0+ kernel) on the node.

2.  Install CRI-O

3.  Configure the K8s Kubelet to use CRI-O

4.  Install the shiftfs kernel module (recommended)

The sub-sections below describe each of these in detail.

### Installing Ubuntu Focal or Bionic

Make sure your host has Ubuntu Focal (20.04) or Ubuntu Bionic (18.04).

The kernel should be >= 5.0. If using Ubuntu Bionic and the kernel is < 5.0, you
can upgrade it with:

```console
$ sudo apt-get update && sudo apt install --install-recommends linux-generic-hwe-18.04 -y
```

### Installing CRI-O on a K8s Worker Node

CRI-O is a light-weight container manager/runtime developed specifically for K8s
(primarily by RedHat). It's a lighter alternative to the heavier containerd.

CRI-O logically sits between K8s and low-level container runtimes such as
Sysbox. When deploying a pod, the K8s control plane sends an instruction to the
Kubelet on a node, and Kubelet sends the instruction to CRI-O. CRI-O then works
with the low-level runtime (e.g., OCI runc or Sysbox) to create the pod.

CRI-O v1.20 introduces support for "rootless pods". That is, launching K8s
pods in which the root user on the pod maps to an unprivileged user on the
host via the Linux user-namespace. This is a key feature required for creating
pods with Sysbox.

Thus, CRI-O v1.20 is required on the K8s worker nodes where Sysbox will be
installed.

*   NOTE: Do **NOT** install CRI-O versions earlier that v1.20, as they don't support
    rootless pods and thus won't work with Sysbox.

*   NOTE: Do **NOT** install CRI-O v1.21 as it has a problem that breaks isolation in
    rootless pods (i.e., maps the pod's root user to the root user on the
    host). We are working to resolve this with the CRI-O developers.

#### CRI-O Installation Steps

The installation steps for CRI-O are here: https://cri-o.io/

For example, to install CRI-O on a Ubuntu-Focal host follow these steps.

As root:

```console
VERSION=1.20
OS=xUbuntu_20.04

echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -

apt-get update
apt-get install cri-o cri-o-runc
```

If all is well, then the CRI-O systemd service unit should be running:

```console
$ systemctl status crio
● crio.service - Container Runtime Interface for OCI (CRI-O)
     Loaded: loaded (/lib/systemd/system/crio.service; disabled; vendor preset: enabled)
     Active: active (running) since Sun 2021-04-25 04:06:52 UTC; 2 days ago
       Docs: https://github.com/cri-o/cri-o
```

Next we will configure CRI-O, and later we will configure K8s to use CRI-O.

#### CRI-O Configuration

In order to use CRI-O with K8s and Sysbox, you must set the following
configurations:

1.  Add an entry for user "containers" to the `/etc/subuid` file on the host. For example:

```console
$ cat /etc/subuid
cesar:100000:65536
containers:165536:268435456
```

Same applies to `/etc/subgid`.

When creating rootless pods, CRI-O will choose the Linux user-namespace ID
mappings from range of user "containers" in these files.

For example, it will map the user ID range \[0:65535] on the pod to the
unprivileged user ID range \[165536:231071] on the host. This way the processes
running in the pod will have permission to access resources inside the pod only.

Notice that since `/etc/subuid` entry for user "containers" has a range of
268435456, CRI-O can map up to (268435456 / 65536) = 4096 pods on that host
without collision.

2.  By default CRI-O uses systemd-managed cgroups for containers. However, K8s
    uses legacy cgroups. Thus, you must either configure CRI-O to use legacy
    cgroups or alternatively configure K8s to use systemd-managed cgroups.

To configure CRI-O to use legacy cgroups, use the following settings in
`/etc/crio/crio.conf`:

```toml
# Cgroup setting for conmon
#conmon_cgroup = "system.slice"
conmon_cgroup = "pod"

# Cgroup management implementation used for the runtime.
#cgroup_manager = "systemd"
cgroup_manager = "cgroupfs"
```

Then restart CRI-O:

```console
$ systemctl restart crio
```

Alternatively, to configure K8s to use systemd-managed cgroups, follow the steps
[here](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/).

Either cgroup mode (legacy or systemd-managed) works with Sysbox.

### Configuring the K8s Kubelet to use CRI-O

Once CRI-O is installed on the Kubernetes worker node, the next step is to
configure the K8s Kubelet to use CRI-O.

Note that starting with K8s v1.20, K8s uses containerd as the default container
runtime (prior to v1.20 it used Docker via the "Dockershim").

The way you do this configuration depends on whether you are initializing K8s on
the worker node for the first time, or whether K8s is already installed and you
want to switch it's runtime config from containerd to CRI-O.

Both of these are described below.

#### Configuring a new K8s worker node with CRI-O

If you are doing a fresh initialization of K8s on a worker node (i.e., you are
joining the worker node to a K8s cluster) then you are typically doing this with
the K8s [kubeadm][kubeadm] tool.

If that's the case, configuring K8s to use CRI-O on the node is trivial. Simply
add the `--cri-socket="/var/run/crio/crio.sock"` option to the `kubeadm join`
command.

For example, assuming you've setup a K8s control plane with:

```console
$ kubeadm init --kubernetes-version=v1.20.2 --pod-network-cidr=10.244.0.0/16
```

then you can initialize a worker node and configure to use CRI-O with:

```console
$ kubeadm token create --print-join-command 2> /dev/nul
$ kubeadm join --cri-socket="/var/run/crio/crio.sock" <join-token-from-prior-command>
```

This assumes that CRI-O is installed on the node, as described [above](#installing-cri-o-on-a-k8s-worker-node).

If all is good, then K8s will see the newly added worker node reach the "Ready"
state within a minute or so.

```console
$ kubectl get nodes
NAME        STATUS   ROLES                  AGE     VERSION
k8s-node1   Ready    control-plane,master   3d19h   v1.20.2
k8s-node2   Ready    <none>                 3d4h    v1.20.2
```

#### Reconfiguring an existing K8s worker node with CRI-O

If you have a K8s worker node that was previously initialized (e.g., a worker
node on a GKE cluster) but you want to switch the container runtime from
containerd to CRI-O, then follow the steps below.

1.  Modify the `KUBELET_OPTS` environment variable defined in the
    `/etc/default/kubelet`. This variable dictates the command line options passed
    to Kubelet when it starts.

The following `sed` commands do this (must run as root):

```console
$ sed -i 's@--container-runtime-endpoint=unix:///run/containerd/containerd.sock@--container-runtime-endpoint=unix:///run/crio/crio.sock@g' /etc/default/kubelet
$ sed -i 's@--runtime-cgroups=/system.slice/containerd.service@--runtime-cgroups=/system.slice/crio.service@g' /etc/default/kubelet
```

2.  Restart the Kubelet systemd service:

```console
$ systemctl restart kubelet
```

3.  Verify the Kubelet is happy:

```console
$ systemctl status kubelet
```

4.  Verify K8s is happy (since kubelet was restarted).

```console
$ kubectl get nodes
```

If all is good, the worker node which we just reconfigured the kubelet should
show up as "Ready" in K8s.

### Installing Shiftfs on a K8s Worker Node

Shiftfs is a kernel module that perform user-ID and group-ID "shifting". It's
needed if you will be mounting host files or directories onto pods created
with K8s + Sysbox (i.e., hostPath volumes).

If you won't be mounting hostPath volumes onto Sysbox pods, shiftfs is not
required.

Shiftfs comes by default in Ubuntu desktop and server versions, but
unfortunately it's not included in cloud versions (e.g., Ubuntu AWS AMIs). You
can run `modinfo shiftfs` on a host to verify it's presence.

If shiftfs is included, then you don't have to do anything else. If it's not
included, you can install it manually.

It's pretty easy to install there is a shiftfs dynamic kernel module (DKMS) here:

https://github.com/toby63/shiftfs-dkms

*   NOTE: shiftfs is only supported on Ubuntu-based kernels at this time. Even if
    you manage to build and install it on non-Ubuntu kernels, Ubuntu carries other
    kernel patches (e.g., overlayfs) that enable proper interaction with
    shiftfs. Without those patches shiftfs won't work well as used by Sysbox.

For example, to manually build and install shiftfs on a Ubuntu-based host with a
5.4 kernel, these are the instructions (as described in the web-page referenced
above):

```console
$ apt-get install -y git make dkms
$ git clone -b k5.4 https://github.com/toby63/shiftfs-dkms.git shiftfs-k54
$ cd shiftfs-k54
$ ./update1
$ sudo make -f Makefile.dkms
```

If all is well, then `modinfo shiftfs` should show that shiftfs is present
on the host:

```console
$ modinfo shiftfs
filename:       /lib/modules/5.4.0-71-generic/kernel/fs/shiftfs.ko
license:        GPL v2
description:    id shifting filesystem
author:         Christian Brauner <christian.brauner@ubuntu.com>
author:         Seth Forshee <seth.forshee@canonical.com>
author:         James Bottomley
...
```

That's it!

Once the K8s node is configured as described above, you can install Sysbox
as shown [here](../README.md#sysbox-installation).

[kubeadm]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
