# Multi-Cluster Kubernetes using Container Ingress Services (CIS) and BIG-IP

Today, organizations are increasingly deploying multiple Kubernetes clusters. Deploying multiple Kubernetes clusters can improve availability, isolation and scalability. This user-guide documents how CIS can automate BIP-IP to provide Edge Ingress services for a dev to prod Kubernetes cluster.

## Multi-Cluster Application Architecture

In this user-guide, each cluster runs a full copy of the application. This simple but powerful approach enables an application to be graduated from dev to prod. Future user-guides will focus on multiple availability zones using health-aware global load balancing. Diagram below represents the two cluster in the user-guide. 

![diagram](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-10_12-07-00.png)

Demo on YouTube [video](https://www.youtube.com/)

### Environment parameters

* Two K8S 1.21 cluster - one master and two worker nodes
* CIS 2.4.0
* AS3: 3.26
* BIG-IP 15.1

## Kubernetes Flannel Modification

Changes are required to the CIDR network flannel on running Kubernetes cluster. Assuming that you have installed a fresh Kubernetes cluster via kubeadm builder tool with adopting appropriate --pod-network-cidr flag in kubeadm init command shown below:

```
kubeadm init --apiserver-advertise-address=192.168.200.70 --pod-network-cidr=10.244.0.0/16
kubeadm init --apiserver-advertise-address=192.168.200.80 --pod-network-cidr=10.245.0.0/16
```

**Note:** A workaround is to edit the ConfigMap file of flannel, kube-flannel-cfg and to replace with the new Network value. Cluster reboot is required. Replace and add the "Network", "VNI" fields under net-conf.json header in the relevant Flannel ConfigMap with a new network IP range:

    $ kubectl edit cm kube-flannel-cfg -n kube-system

```
net - conf.json: | {
	"Network": "10.245.0.0/16",
	"Backend": {
		"Type": "vxlan",
		"VNI": 11
	}
}
```
Output should look like the example below for the prod cluster

```
[kube@ks8-prod-master pod-deployment]$ cat /run/flannel/subnet.env
FLANNEL_NETWORK=10.245.0.0/16
FLANNEL_SUBNET=10.245.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```
Changes are also required to all the backend-data annotations of the Kubenetes nodes in the cluster. This includes the big-ip-node documented below

    $ kubectl edit nodes

```
annotations:
      flannel.alpha.coreos.com/backend-data: '{"VNI":11,"VtepMAC":"ae:0b:f0:f4:fc:1a"}'
      flannel.alpha.coreos.com/backend-type: vxlan
      flannel.alpha.coreos.com/kube-subnet-manager: "true"
      flannel.alpha.coreos.com/public-ip: 192.168.200.80
```

## BIG-IP Vxlan Tunnels and Self-IPs Setup

**Note:** Multi-cluster requires that the vxlan tunnels and self-IPs are created in a unique tenant. In this example i have create two tenant names dev and prod. Creating the vxlan tunnels and self-IPs in the /Common tenant wont work.

### Create vxlan tunnels and self-IPs for the dev cluster

* create net tunnels vxlan fl-vxlan port 8472 flooding-type none
* create net tunnel tunnel vxlan-tunnel-dev key 1 profile fl-vxlan local-address 192.168.200.60
* create net self 10.244.20.60 address 10.244.20.60/255.255.0.0 allow-service none vlan vxlan-tunnel-dev

### Create vxlan tunnels and self-IPs for the prod cluster

* create net tunnel tunnel vxlan-tunnel-prod key 11 profile fl-vxlan local-address 192.168.200.60
* create net self 10.245.20.60 address 10.245.20.60/255.255.0.0 allow-service none vlan vxlan-tunnel-prod

### Example of vxlan tunnels and Self-IPs for the vxlan-tunnel-dev

Tunnel profile fl-vxlan configuration for vxlan-tunnel-dev

![vxlan-tunnel](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-10_12-11-17.png)

Self-IPs configuration for vxlan-tunnel-dev

![self-ip](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-10_12-28-33.png)

### Example of vxlan tunnels and Self-IPs for the vxlan-tunnel-prod

Tunnel profile fl-vxlan configuration for vxlan-tunnel-prod

![vxlan-tunnel](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-10_12-11-53.png)

Self-IPs configuration for vxlan-tunnel-prod

![self-ip](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-10_12-27-39.png)

### Create BIG-IP Node required Tunnel: vxlan-tunnel-dev and vxlan-tunnel-prod

Find the VTEP MAC address for tunnels vxlan-tunnel-dev and vxlan-tunnel-prod.

```
(tmos)# show net tunnels tunnel vxlan-tunnel-dev all-properties

-------------------------------------------------
Net::Tunnel: vxlan-tunnel-dev
-------------------------------------------------
MAC Address                     00:50:56:bb:32:66
Interface Name                    vxlan-tunnel-~1

Incoming Discard Packets                        0
Incoming Error Packets                          0
Incoming Unknown Proto Packets                  0
Outgoing Discard Packets                        0
Outgoing Error Packets                         10
HC Incoming Octets                              0
HC Incoming Unicast Packets                     0
HC Incoming Multicast Packets                   0
HC Incoming Broadcast Packets                   0
HC Outgoing Octets                              0
HC Outgoing Unicast Packets                     0
HC Outgoing Multicast Packets                   0
HC Outgoing Broadcast Packets                   0
```

```
tmos)# show net tunnels tunnel vxlan-tunnel-prod all-properties

-------------------------------------------------
Net::Tunnel: vxlan-tunnel-prod
-------------------------------------------------
MAC Address                     00:50:56:bb:32:66
Interface Name                    vxlan-tunnel-~2

Incoming Discard Packets                        0
Incoming Error Packets                          0
Incoming Unknown Proto Packets                  0
Outgoing Discard Packets                        0
Outgoing Error Packets                         10
HC Incoming Octets                              0
HC Incoming Unicast Packets                     0
HC Incoming Multicast Packets                   0
HC Incoming Broadcast Packets                   0
HC Outgoing Octets                              0
HC Outgoing Unicast Packets                     0
HC Outgoing Multicast Packets                   0
HC Outgoing Broadcast Packets                   0
```

### Create two “dummy” Kubernetes Node for Tunnel: vxlan-tunnel-dev and vxlan-tunnel-prod

Include all of the flannel Annotations. Define the backend-data and public-ip Annotations with data from the BIG-IP VXLAN:

**vxlan-tunnel-dev**
```
apiVersion: v1
kind: Node
metadata:
  name: vxlan-tunnel-dev
  annotations:
    #Replace MAC with your BIGIP Flannel VXLAN Tunnel MAC
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"00:50:56:bb:32:66"}'
    flannel.alpha.coreos.com/backend-type: "vxlan"
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    #Replace IP with Self-IP for your deployment
    flannel.alpha.coreos.com/public-ip: "192.168.200.60"
spec:
  #Replace Subnet with your BIGIP Flannel Subnet
  podCIDR: "10.244.20.0/24
```

**vxlan-tunnel-prod**
```
apiVersion: v1
kind: Node
metadata:
  name: vxlan-tunnel-prod
  annotations:
    #Replace MAC with your BIGIP Flannel VXLAN Tunnel MAC
    flannel.alpha.coreos.com/backend-data: '{"VNI":11,"VtepMAC":"00:50:56:bb:32:66"}'
    flannel.alpha.coreos.com/backend-type: "vxlan"
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    #Replace IP with Self-IP for your deployment
    flannel.alpha.coreos.com/public-ip: "192.168.200.60"
spec:
  #Replace Subnet with your BIGIP Flannel Subnet
  podCIDR: "10.245.20.0/24"
```

dev-cluster
* f5-bigip-dev-node.yaml [repo](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/dev-cluster/cis/cis-deployment/f5-bigip-dev-node.yaml)

prod-cluster
* f5-bigip-prod-node.yaml [repo](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/prod-cluster/cis/cis-deployment/f5-bigip-prod-node.yaml)


## Deploy CIS for each BIG-IP for dev-cluster and prod-cluster

Configuration options available in the CIS controller
```
    spec: 
      containers: 
        - 
          args: 
            - "--bigip-username=$(BIGIP_USERNAME)"
            - "--bigip-password=$(BIGIP_PASSWORD)"
            - "--bigip-url=192.168.200.60"
            - "--bigip-partition=dev"
            - "--namespace=dev"
            - "--pool-member-type=cluster"
            - "--flannel-name=/dev/vxlan-tunnel-dev"
            - "--log-level=DEBUG"
            - "--insecure=true"
            - "--log-as3-response=true"
            - "--custom-resource-mode=true"
            - "--ipam=true"
```

Configuration options available in the CIS controller
```
    spec: 
      containers: 
        - 
          args: 
            - "--bigip-username=$(BIGIP_USERNAME)"
            - "--bigip-password=$(BIGIP_PASSWORD)"
            - "--bigip-url=192.168.200.60"
            - "--bigip-partition=prod"
            - "--namespace=prod"
            - "--pool-member-type=cluster"
            - "--flannel-name=/prod/vxlan-tunnel-prod"
            - "--log-level=DEBUG"
            - "--insecure=true"
            - "--log-as3-response=true"
            - "--custom-resource-mode=true"
            - "--ipam=true"
          command: 
```

dev-cluster
* f5-cis-deployment.yaml [repo](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/dev-cluster/cis/cis-deployment/f5-cis-deployment.yaml)

prod-cluster
* f5-cis-deployment.yaml [repo](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/prod-cluster/cis/cis-deployment/f5-cis-deployment.yaml)