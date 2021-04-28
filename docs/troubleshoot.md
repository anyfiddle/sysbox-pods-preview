# K8s Node Troubleshooting

## Pod stuck in "Creating" status

If you are deploying a pod with Sysbox and it gets stuck in the "Creating"
state for a long time (e.g., > 1 minute), then something is likely wrong.

To debug, start by checking the CRI-O and Kubelet logs on the node where
the pod was scheduled:

```console
journalctl -eu crio
journalctl -eu kubelet
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
$ journalctl -eu sysbox
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
