# K8s Node Troubleshooting

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
