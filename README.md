# Context

This install is using 4 VMs, with static IPs:
 - 3 controlplanes
 - 1 worker

All those VMs are using virtualbox, configured to have their 1st NIC bridged on the host network, with the parameter `promiscuous=Allow All`.

They use PXE. (iPXE config is out of this scope)

The kubernetes endpoint (which will be used to talk to the API server) will be an highly available IP, shared between controlplane nodes using a L2 announcement.

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

Those 3 files can be regenerated at any time.

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


# Get kubectl config

Save the `kubectl` config file into `./kubeconfig`:

    talosctl kubeconfig ./kubeconfig -n 192.168.42.210

From now, if the env var `KUBECONFIG` exists and targets this file, `kubectl` (and other clients like `k9s`) will target the `vtalos` cluster.

    export KUBECONFIG=$PWD/kubeconfig


# Install cilium

At this stage, nodes aren't ready because they lack a CNI. Notably, they won't be able to run some user workloads (pods).

We need to install `cilium` to make them ready.

First, install `helm` client, then run:

```
helm template \
    cilium \
    cilium/cilium \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
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
> helm-cilium.yaml
```

Then run:

    kubectl apply -f helm-cilium.yaml
    # wait a bit
    cilium status

When cilium status is all OK (fully green), you can deploy some other cilium objects, that will be use by our demo application (`rello`):

    kubectl apply -f cilium -R


From now, you can use `hubble`, the observability layer  of `cilium`, by running:

    cilium hubble ui

This will relay your local port 12000 to the huble UI service and will open your browser to this URL: http://127.0.0.1:12000


# The rello app

Run:

    kubectl apply -f rello -R

Test it using `curl` and the various endpoints we created (LB, ingress, NodePort)

Example:

    curl -i http://42.42.42.43/app/hello?name=foo

The result should look like:

    HTTP/1.1 200 OK
    date: Mon, 28 Oct 2024 14:30:03 GMT
    server: envoy
    content-length: 62
    content-type: application/json
    x-envoy-upstream-service-time: 1
    
    {"msg":"Hello Foo","views":1,"host":"rello-b6d567c5c-dc4sj"}

with the `views` field incremented by each call (that's why we use redis).

Notice that the `server: envoy` header implies we reached the demo app via an ingress provided by `cilium`: `envoy`. Using a pure LoadBalancer or NodePort service should reveal another value for this header.

# Security considerations

In the rello namespace, we enforce 2 security features:

 - PSA: Pod Security Admission
 - NetworkPolicies

## Pod Security Admission

PSA deprecated the old PSP (PodSecurityPolicy). Basically it defines 3 levels:

 - privileged: a pod can do anything it likes: priv escalation, host network, run as root, ....
 - baseline: this level enforces some basic limits: no host network, drops of some capabilities, ...
 - restricted: a pod is very restricted

A namespace can set this level and a policy among:

 - warning: accept the pod but output a warning 
 - audit: accept the pod but log a warning
 - enforce: refuse the pod

For the rello namespace, we chose to enforce the `baseline` level. See https://kubernetes.io/docs/concepts/security/pod-security-admission/ for more details.


## Network Policies

See https://kubernetes.io/docs/concepts/services-networking/network-policies/

In rello, we chose to have a DENY ALL policy by default. No ingress nor egress traffic is allowed inside this namespace. Then, we allowed the strict minimum policies for the app to behave correctly:

 - ingress rello from any, tcp/8000
 - egress from rello to kube-system/coredns, tcp/53 and udp/53
 - egress from rello to redis, tcp/6379
 - ingress redis from rello, tcp/6379

The last 2 policies seem to be very redundant, but it's mandatory since we chose deny by default both for ingress & egress traffic. Other strategies might be chosen.

See `rello/np-*.yaml` for details.

