# K8s Node Troubleshooting

## Contents

*   [The sysbox-deploy-k8s daemonset causes pods to enter "Error" state](#the-sysbox-deploy-k8s-daemonset-causes-pods-to-enter-error-state)
*   [CRI-O Can't find CNI Binaries](#cri-o-cant-find-cni-binaries)
*   [Pod stuck in "Creating" status](#pod-stuck-in-creating-status)
*   [Sysbox health status](#sysbox-health-status)
*   [CRI-O health status](#cri-o-health-status)
*   [Kubelet health status](#kubelet-health-status)
*   [Low-level Debug with crictl](#low-level-debug-with-crictl)

## The sysbox-deploy-k8s daemonset causes pods to enter "Error" state

During [Sysbox installation](../README.md#sysbox-installation), one of the steps
is to run the sysbox-deploy-k8s daemonset via:

```console
$ kubectl apply -f https://raw.githubusercontent.com/nestybox/sysbox-pods-preview/master/k8s-manifests/rbac/sysbox-rbac.yaml
$ kubectl apply -f https://raw.githubusercontent.com/nestybox/sysbox-pods-preview/master/k8s-manifests/daemonset/sysbox-deploy-k8s.yaml
```

The sysbox-deploy-k8s daemonset configures and restarts the CRI-O container
runtime. This is necessary for CRI-O to pick up the fact that Sysbox is present
on the host.

Unfortunately this causes all pods on the Kubernetes worker node(s) where the
daemonset is running to be restarted. As a result, you may see something like
this immediately after running the daemonset:

```console
$ kubectl get all --all-namespaces
NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-74ff55c5b-ff58f             1/1     Running   0          11d
kube-system   pod/coredns-74ff55c5b-t6t5b             1/1     Error     0          11d
kube-system   pod/etcd-k8s-node2                      1/1     Running   1          11d
kube-system   pod/kube-apiserver-k8s-node2            1/1     Running   1          11d
kube-system   pod/kube-controller-manager-k8s-node2   1/1     Running   1          11d
kube-system   pod/kube-flannel-ds-4pqgp               1/1     Error     4          4d22h
kube-system   pod/kube-flannel-ds-lkvnp               1/1     Running   0          10d
kube-system   pod/kube-proxy-4mbfj                    1/1     Error     4          4d22h
kube-system   pod/kube-proxy-5lfz4                    1/1     Running   0          11d
kube-system   pod/kube-scheduler-k8s-node2            1/1     Running   1          11d
kube-system   pod/sysbox-deploy-k8s-rbl76             1/1     Error     1          136m
```

This is a condition that should resolve itself within 30 secs to 1 minute after
running the sysbox-deploy-k8s daemonset, as CRI-O will restart and Kubernetes
will automatically restart the affected pods.

If the sysbox-deploy-k8s daemonset ran successfully, you should see the following:

1.  The installation daemonset will add the label `sysbox-runtime=running` to
    nodes where Sysbox was installed:

```console
$ kubectl get nodes --show-labels
NAME        STATUS   ROLES                  AGE     VERSION   LABELS
k8s-node1   Ready    control-plane,master   3d21h   v1.20.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=
k8s-node2   Ready    <none>                 3d6h    v1.20.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node2,kubernetes.io/os=linux,name=sysbox-deploy,sysbox-runtime=running
```

2.  The sysbox-deploy-k8s logs should show the following:

```console
$ kubectl -n kube-system logs pod/sysbox-deploy-k8s-rbl76
Sysbox is running on the node.
Done.
```

Those are the logs from the sysbox-deploy-k8s daemonset after it's restarted by
Kubernetes (since it also entered the error state when CRI-O was restarted).

In addition, the "previous" logs for the sysbox-deploy-k8s daemonset should show:

```console
$ kubectl -n kube-system logs -p pod/sysbox-deploy-k8s-rbl76
Installing Sysbox on host
Detected host distro: ubuntu_20.04
Configuring host sysctls
kernel.unprivileged_userns_clone = 1
fs.inotify.max_queued_events = 1048576
fs.inotify.max_user_watches = 1048576
fs.inotify.max_user_instances = 1048576
kernel.keys.maxkeys = 20000
kernel.keys.maxbytes = 400000
Probing kernel modules

Configfs kernel module is not loaded. Configfs may be required by certain applications running inside a Sysbox container.

Starting Sysbox
Adding Sysbox to CRI-O config
Adding K8s label "sysbox-runtime=running" to node
node/k8s-node3 labeled
Sysbox installation completed.
Restarting CRI-O (this will temporarily disrupt all pods on the K8s node (for up to 1 minute)).
/opt/sysbox/scripts/sysbox-deploy-k8s.sh: line 221:    71 Terminated              systemctl restart crio
```

Those logs show the original execution of the daemonset, up to when it restarted
CRI-O (at which point the daemonset pod was terminated and later restarted).

If for some reason you do not see these logs, please [contact us](../README.md#contact)
and we will be happy to help.

## CRI-O Can't find CNI Binaries

If the CRI-O log (`journalctl -u crio`) shows an error such as:

```console
Error validating CNI config file /etc/cni/net.d/10-containerd-net.conflist: [failed to find plugin in opt/cni/bin]
```

it means it can't find the binaries for the CNI. CRI-O looks for those in `/opt/cni/bin`.

This error can occur when you install CRI-O before K8s on the node (i.e., the CRI-O installation
does not include the CNIs). Once you install K8s on the node and restart CRI-O, the error
should not longer appear (because K8s does install several CNIs).

If the error persists after installing K8s, sometimes it's because K8s installed
the CNI binaries in a different directory than what CRI-O expects. For example,
on a GKE node K8s installs the CNI binaries at `/home/kubernetes/bin/`.

To solve this, set the following configuration on the `/etc/crio/crio.conf` file
and restart CRI-O:

```console
plugin_dirs = [
        "/opt/cni/bin/",
        "/home/kubernetes/bin/"
]
```

After restarting CRI-O, it should pick the CNI binaries correctly.

## Pod stuck in "Creating" status

If you are deploying a pod with Sysbox and it gets stuck in the "Creating"
state for a long time (e.g., > 1 minute), then something is likely wrong.

To debug, start by checking the CRI-O and Kubelet logs on the node where
the pod was scheduled:

```console
$ journalctl -eu crio
$ journalctl -eu kubelet
```

These often have information that helps pin-point the problem.

If the kubelet log shows an error such as `failed to find runtime handler sysbox-runc from runtime list`,
then this means CRI-O has not recognized Sysbox for some reason.

In this case, double check that the CRI-O config has the Sysbox runtime
directive in it (the sysbox-deploy-k8s daemonset should have set this config):

```console
$ cat /etc/crio/crio.conf

...

[crio.runtime.runtimes.sysbox-runc]
allowed_annotations = ["io.kubernetes.cri-o.userns-mode"]
runtime_path = "/usr/bin/sysbox-runc"
runtime_type = "oci"

...
```

If the sysbox runtime config is present, then try restarting CRI-O on the worker node:

```console
systemctl restart crio
```

Note that restarting CRI-O will cause all pods on the node to be restarted
(including the kube-proxy and CNI pods).

If the sysbox runtime config is not present, then [uninstall](../README.md#sysbox-uninstallation)
and re-install\[../README.md#sysbox-installation) the sysbox daemonset.

## Sysbox health status

```console
$ systemctl status sysbox
$ systemctl status sysbox-mgr
$ systemctl status sysbox-fs

$ journalctl -eu sysbox
$ journalctl -eu sysbox-mgr
$ journalctl -eu sysbox-fs
```

## CRI-O health status

```console
$ systemctl status crio
$ journalctl -eu crio
```

## Kubelet health status

```console
$ systemctl status kubelet
$ journalctl -eu kubelet
```

## Low-level Debug with crictl

The `crictl` tool can be used to communicate with CRI implementations directly
(e.g., CRI-O or containerd). `crictl` is typically present in K8s nodes. If for
some reason it's not, you can install it as shown here:

https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md

You should install `crictl` on the K8s worker nodes where CRI-O is installed,
and configure it as follows:

```console
$ cat /etc/crictl.yaml
runtime-endpoint: "unix:///var/run/crio/crio.sock"
image-endpoint: "unix:///var/run/crio/crio.sock"
timeout: 0
debug: false
pull-image-on-create: true
disable-pull-on-run: false
```

The key is to set `runtime-endpoint` and `image-endpoint` to CRI-O as shown
above.

After this, you can use crictl to determine the health of the pods on the node:

For example, in a properly initialized K8s worker node you should see the
kube-proxy pod running on the node.

```console
$ sudo crictl ps
CONTAINER ID        IMAGE                                                                                             CREATED             STATE               NAME                ATTEMPT             POD ID
48912b06799d2       43154ddb57a83de3068fe603e9c7393e7d2b77cb18d9e0daf869f74b1b4079c0                                  2 days ago          Running             kube-proxy          40                  91462637d4e23
```
