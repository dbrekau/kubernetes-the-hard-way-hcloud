# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [data center](https://docs.hetzner.com/general/others/data-centers-and-connection/), ensured on different hosts. We will create everything in the data center located in Nuremberg, Germany (nbg1) in this tutorial, but you may choose a different one.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Hetzner Cloud Network

In this section a [Hetzner Cloud Network](https://docs.hetzner.com/cloud/networks/getting-started/creating-a-network) will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` network:

```
hcloud network create --name kubernetes-the-hard-way --ip-range 10.240.0.0/16
hcloud network add-subnet kubernetes-the-hard-way --ip-range 10.240.0.0/24 --network-zone eu-central --type cloud
```

> A network with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster is needed.

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

Create a firewall:

```
hcloud firewall create --name kubernetes-the-hard-way-allow-external
```

Allow SSH, ICMP, and HTTPS:

```
hcloud firewall add-rule kubernetes-the-hard-way-allow-external \
  --direction in --description ICMP --protocol icmp --source-ips 0.0.0.0/0 --source-ips ::/0
hcloud firewall add-rule kubernetes-the-hard-way-allow-external \
  --direction in --description ICMP --protocol tcp --port 22 --source-ips 0.0.0.0/0 --source-ips ::/0
hcloud firewall add-rule kubernetes-the-hard-way-allow-external \
  --direction in --description ICMP --protocol tcp --port 6443 --source-ips 0.0.0.0/0 --source-ips ::/0
```

### Placement Group

A [Placement Group](https://docs.hetzner.com/cloud/placement-groups/overview) is used to ensure the VMs are running on different hosts for high availability.

```
hcloud placement-group create --name kubernetes-the-hard-way --type spread
```

### Load Balancer

An [external load balancer](https://www.hetzner.com/cloud/load-balancer) will be used to expose the Kubernetes API Servers to remote clients.

```
hcloud load-balancer create --location nbg1 --name kubernetes-the-hard-way-api --type lb11
```

> output

```
LoadBalancer XXXXXX created
IPv4: XXX.XXX.XXX.XXX
IPv6: XXXX:XXXX:XXXX:XXXX::1
```

> Note these addresses or show again with `hcloud load-balancer list`.

Attach the Load Balancer to our created network.

```
hcloud load-balancer attach-to-network --network kubernetes-the-hard-way --ip 10.240.0.254 kubernetes-the-hard-way-api
```

## Add SSH key

SSH will be used to configure the controller and worker instances. You'll need to have a key pair on your machine. We assume that you have an ED25519 key located at `~/.ssh/id_ed25519`, and the public part at `~/.ssh/id_ed25519.pub`.

Add your SSH key:

```
hcloud ssh-key create --name kubernetes-the-hard-way --public-key-from-file ~/.ssh/id_ed25519.pub
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 20.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  hcloud server create --name controller-${i} \
    --location nbg1 \
    --type cx21 \
    --placement-group kubernetes-the-hard-way \
    --image ubuntu-20.04 \
    --network kubernetes-the-hard-way \
    --ssh-key 'kubernetes-the-hard-way' \
    --label 'kubernetes-the-hard-way=' \
    --label 'controller='
done
```

### Kubernetes Workers

TBD pod subnet

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  hcloud server create --name worker-${i} \
    --location nbg1 \
    --type cx21 \
    --placement-group kubernetes-the-hard-way \
    --image ubuntu-20.04 \
    --network kubernetes-the-hard-way \
    --ssh-key 'kubernetes-the-hard-way' \
    --label 'kubernetes-the-hard-way=' \
    --label 'worker='
done
```

### Verification

List the compute instances:

```
hcloud server list -l kubernetes-the-hard-way
```

> output

```
ID         NAME           STATUS    IPV4              IPV6                       DATACENTER
XXXXXXXX   controller-0   running   XXX.XXX.XXX.XXX   XXXX:XXXX:XXXX:XXXX::/64   nbg1-dcX
XXXXXXXX   controller-1   running   XXX.XXX.XXX.XXX   XXXX:XXXX:XXXX:XXXX::/64   nbg1-dcX
XXXXXXXX   controller-2   running   XXX.XXX.XXX.XXX   XXXX:XXXX:XXXX:XXXX::/64   nbg1-dcX
XXXXXXXX   worker-0       running   XXX.XXX.XXX.XXX   XXXX:XXXX:XXXX:XXXX::/64   nbg1-dcX
XXXXXXXX   worker-1       running   XXX.XXX.XXX.XXX   XXXX:XXXX:XXXX:XXXX::/64   nbg1-dcX
XXXXXXXX   worker-2       running   XXX.XXX.XXX.XXX   XXXX:XXXX:XXXX:XXXX::/64   nbg1-dcX
```

> Please note this addresses, you'll need them.

## Configuring SSH Access

### SSH config

A SSH config is needed for later parts in this tutorial, which we will now create.

hcloud server list -l kubernetes-the-hard-way -o noheader -o columns=name,ipv4 | while read server; do
tee -a config << END
Host $(echo $server | awk '{print $1}')
    User root
    Hostname $(echo $server | awk '{print $2}')
END
done

> This will fetch all Servernames and their IP addresses and goes through it line by line. It then creates a ssh config entry for each.

### Test SSH access

Test SSH access to the `controller-0` compute instances:

```
ssh -F config controller-0
```

You should be logged into the `controller-0` instance:

```
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-1042-gcp x86_64)
...
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```
root@controller-0:~# exit
```

> output

```
logout
Connection to XX.XX.XX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
