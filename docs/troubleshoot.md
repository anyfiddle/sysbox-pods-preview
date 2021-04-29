# K8s Node Troubleshooting

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

After retarting CRI-O, it should pick the CNI binaries correctly.

## Sysbox-Deploy-K8s Daemonset Fails

If the sysbox-deploy-k8s daemonset fails, check it's logs via:

```console
$ kubectl -n kube-system logs <sysbox-deploy-k8s-pod>
```

Normally you should see something like this:

```
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

Shiftfs kernel module is not loaded. Shiftfs is required for host volume mounts into Sysbox containers to have proper ownership (user-ID and group-ID).

Starting Sysbox
Adding Sysbox to CRI-O config
Adding K8s label "sysbox-runtime=running" to node
node/gke-cluster-1-sysbox-pool-f42ba242-k0zt labeled
Sysbox installation completed.
Signaling CRI-O to reload it's config.
```

If the logs show an error, then there is a problem. Please [contact us](../README.md#contact) so
we can help you resolve it.

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

```
systemctl restart crio
```

Note that restarting CRI-O will cause all pods on the node to be restarted
(including the kube-proxy and CNI pods).

If the sysbox runtime config is not present, then [uninstall](../README.md#sysbox-uninstallation)
and re-install[../README.md#sysbox-installation) the sysbox daemonset.

### Sysbox health status

```console
$ systemctl status sysbox
$ systemctl status sysbox-mgr
$ systemctl status sysbox-fs

$ journalctl -eu sysbox
$ journalctl -eu sysbox-mgr
$ journalctl -eu sysbox-fs
```

### CRI-O health status

```console
$ systemctl status crio
$ journalctl -eu crio
```

### Kubelet health status

```console
$ systemctl status kubelet
$ journalctl -eu kubelet
```

### crictl

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
