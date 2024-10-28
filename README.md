# Context

This install is using 4 VMs, with static IPs:
 - 3 controlplane
 - 1 worker

All those VMs are using virtualbox, configured to have their 1st NIC bridged on the host network, with the parameter `promiscuous=Allow All`.

They use PXE.

The endpoint (which will be used to talk to the API server) will be an highly available IP, shared between controlplane nodes using a L2 technique.

Finally, we will use `cilium` as CNI, with kube-proxy replacement mode, with `hubble` enabled (obersvability, service-mesh) and with L2 (arp) and L3 (bgp) announcement capabilities.


# Generate secrets

Run:

    talosctl gen secrets -o secrets.yaml

This file will contain the secrets that are needed by various parts of the cluster. Remember, `talosctl` and the cluster, when bootstraped, use mutual TLS to maximize security.

Do not loose this file, otherwise you won't be able to manage your cluster.

Do not store it to git as it contains secrets. Use a vault instead.

# Generate configs

Choose a future IP for the endpoint, in the same network as the host one. Say 192.168.42.210.

Now run: (use the schedule-on-controlplanes patch only if you want to save some resource and avoid having dedicated workers)

    talosctl gen config --with-secrets secrets.yaml --config-patch @cilium-no-kube-proxy.patch --config-patch @schedule-on-controlplanes.patch vtalos https://192.168.42.210:6443

It will generate 3 files:

 - talosconfig
 - controlplane.yaml
 - worker.yaml

Do not loose `talosconfig`, otherwise you'll be unable to manage your cluster.

`controlplane.yaml` and `worker.yaml` can be regenerated.

`vtalos` is the name of the cluster, and is called `context` by Talos.

From now, every command that needs a `--talosconfig <file>` (when not using the `--insecure` flag) can omit this flag if the `TALOSCONFIG` env var exists.

    export TALOSCONFIG=$PWD/talosconfig


Inject the endpoints to the config:

    talosctl config endpoint 192.168.42.201 192.168.42.202 192.168.42.203

Those endpoints are the IPs of the controlplane nodes, _not_ the kubernetes endpoint we chose earlier (otherwise the `etcd` cluster will not work correctly).

# Patch nodes config files

Before bootstraping the cluster, we want to `patch` config files with some particular settings for each node. Notably, we want static IP addressing, and a shared, L2 VIP for the endpoint as we mentionned above.

Example of patch for a controlplane node:

```
cat talos-cp-1.patch 
machine:
  network:
    hostname: talos-cp-1
    nameservers:
      - 192.168.42.2
    interfaces:
      - interface: enp0s17
        addresses:
          - 192.168.42.201/24
        vip:
          ip: 192.168.42.210
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.42.1
```

Then, run:

    talosctl machineconfig patch controlplane.yaml --patch @talos-cp-1.patch --output talos-cp-1.yaml


This will generate a talos-cp-1.yaml, derived from controlplane.yaml, including all the specifics from the patch file. Do the same for the 2 other controlplane nodes.

Then, do the same for workers nodes. Here, we want static IP adressing as well, but not the shared VIP.

```
machine:
  network:
    hostname: talos-wo-1
    nameservers:
      - 192.168.42.2
    interfaces:
      - interface: enp0s3
        addresses:
          - 192.168.42.204/24
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.42.1
```

Then, run:

    talosctl machineconfig patch worker.yaml --patch @talos-wo-1.patch --output talos-wo-1.yaml


# Install the nodes

In this section, we assume each node to be installed has booted Talos OS successfully, using PXE for example, and is waiting for its config.

Install the 1st controlplane node:

    talosctl -n 192.168.42.142 apply-config --file talos-cp-1.yaml --insecure

A few remarks:
 - the `insecure` flag instructs talosctl to reach the node directly using the TCP port 50000. This is mandatory as long as the cluster is not formed yet.
 - At boot time, the node was configured to use 192.168.42.142 using DHCP, but as soon as the config is loaded, its IP will change.

Once the 1st controlplane node is ready, it's time to bootstrap the cluster. This needs to be run only once !

    talosctl -n 192.168.42.201 bootstrap 

Then, install the other controlplanes and workers using `apply-config`. Do not run `bootstrap` again !


# Install cilium

At this stage, nodes aren't ready because they lack a CNI. Notably, they won't be able to run some user workloads (pods).

We need to install `cilium` to make them ready.

First, install `helm` client then run:

```
helm template \
    cilium \
    cilium/cilium \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
    # replace kube-proxy \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445 \
    --set ingressController.enabled=true \
    --set ingressController.loadbalancerMode=dedicated \
    --set l2announcements.enabled=true \
    --set k8sClientRateLimit.qps=100 \
    --set k8sClientRateLimit.burst=120 \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true \
    --set bgpControlPlane.enabled=true \
> cilium/helm-resources.yaml
```

Then run:

    kubectl apply -f cilium/helm-resources.yaml
    kubectl apply -f cilium/cilium*.yaml


# The rello app

Run:

    kubectl apply -f rello/*.yaml

Test it using `curl` and the various endpoints we created (LB, ingress, NodePort)


