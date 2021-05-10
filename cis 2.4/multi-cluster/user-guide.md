# Multi-Cluster Kubernetes using Container Ingress Services (CIS) and BIG-IP

Today, organizations are increasingly deploying many more Kubernetes clusters. Deploying multiple Kubernetes clusters can improve availability, isolation and scalability. This user-guide documents two Kubernetes clusters using CIS to automates BIP-IP to provide Edge Ingress services for both clusters.

## Multi-Cluster Application Architecture

In this user-guide, each cluster runs a full copy of the application. This simple but powerful approach enables an application to be graduated from dev to prod. Future user-guides will focus on multiple availability zones or data centers and user traffic routed to the closest or most appropriate cluster using health-aware global load balancer.

![diagram](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-10_12-07-00.png)

Demo on YouTube [video](https://www.youtube.com/)

### Environment parameters

* Multiple K8S 1.21 custer - one master and two worker nodes
* CIS 2.4.0
* AS3: 3.26
* BIG-IP 15.1

## Create Vxlan tunnels and self-IPs for the prod cluster

* create net tunnels vxlan fl-vxlan port 8472 flooding-type none
* create net tunnel tunnel vxlan-tunnel-prod key 11 profile fl-vxlan local-address 192.168.200.60
* create net self 10.245.20.60 address 10.245.20.60/255.255.0.0 allow-service none vlan vxlan-tunnel-prod

## Create Vxlan tunnels and self-IPs for the dev cluster

* create net tunnel tunnel vxlan-tunnel-dev key 1 profile fl-vxlan local-address 192.168.200.60
* create net self 10.244.20.60 address 10.244.20.60/255.255.0.0 allow-service none vlan vxlan-tunnel-dev

## Example of Vxlan tunnels and Self-IPs for the vxlan-tunnel-prod

Tunnel profile fl-vxlan configuration for vxlan-tunnel-prod

![vxlan-tunnel](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-10_12-11-53.png)

Self-IPs configuration for vxlan-tunnel-prod

![self-ip](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-10_12-27-39.png)

## Example of Vxlan tunnels and Self-IPs for the vxlan-tunnel-dev

Tunnel profile fl-vxlan configuration for vxlan-tunnel-dev

![vxlan-tunnel](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-10_12-11-17.png)

Self-IPs configuration for vxlan-tunnel-dev

![self-ip](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-10_12-28-33.png)


## Create BIG-IP Node (vxlan)

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

root@(big-ip-ve6-pme)(cfg-sync Standalone)(Active)(/Common)(tmos)#

```

## Create two “dummy” Kubernetes Node for each vxlan tunnel

Include all of the flannel Annotations. Define the backend-data and public-ip Annotations with data from the BIG-IP VXLAN:

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

```
apiVersion: v1
kind: Node
metadata:
  name: vxlan-tunnel-prod
  annotations:
    #Replace MAC with your BIGIP Flannel VXLAN Tunnel MAC
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"00:50:56:bb:32:66"}'
    flannel.alpha.coreos.com/backend-type: "vxlan"
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    #Replace IP with Self-IP for your deployment
    flannel.alpha.coreos.com/public-ip: "192.168.200.60"
spec:
  #Replace Subnet with your BIGIP Flannel Subnet
  podCIDR: "10.245.20.0/24
```

**Note: Second node create a unique podCIDR**

* f5-bigip-node-91.yaml [repo](https://github.com/mdditt2000/kubernetes-1-20/blob/main/cis%202.4/ha-cluster/big-ip-91/f5-bigip-node-91.yaml)
* f5-bigip-node-92.yaml [repo](https://github.com/mdditt2000/kubernetes-1-20/blob/main/cis%202.4/ha-cluster/big-ip-92/f5-bigip-node-92.yaml)






## Deploy CIS for each BIG-IP

Configuration options available in the CIS controller
```
    spec: 
      containers: 
        - 
          args: 
            - "--bigip-username=$(BIGIP_USERNAME)"
            - "--bigip-password=$(BIGIP_PASSWORD)"
            - "--bigip-url=192.168.200.91"
            - "--bigip-partition=k8s"
            - "--namespace=default"
            - "--pool-member-type=cluster"
            - "--flannel-name=fl-vxlan"
            - "--log-level=DEBUG"
            - "--insecure=true"
            - "--log-as3-response=true"
            - "--custom-resource-mode=true"
          command: 
```

bigip1
* f5-bigip-ctlr-deployment-91.yaml [repo](https://github.com/mdditt2000/kubernetes-1-20/blob/main/cis%202.4/ha-cluster/big-ip-91/f5-bigip-ctlr-deployment-91.yaml)

bigip2
* f5-bigip-ctlr-deployment-92.yaml [repo](https://github.com/mdditt2000/kubernetes-1-20/blob/main/cis%202.4/ha-cluster/big-ip-92/f5-bigip-ctlr-deployment-92.yaml)

Flannel changes

```
Changes required in k8s v1.20 setup for modifying VNI 11 and also vxlan subnet :
 
1. Modify in flannel-kube-cfg configmap file the vlxan subnet and also VNI.
apiVersion: v1
data:
cni-conf.json: |
{
"name": "cbr0",
"cniVersion": "0.3.1",
"plugins": [
{
"type": "flannel",
"delegate": {
"hairpinMode": true,
"isDefaultGateway": true
}
},
{
"type": "portmap",
"capabilities": {
"portMappings": true
}
}
]
}
net-conf.json: |
{
"Network": "10.245.0.0/16",
"Backend": {
"Type": "vxlan",
"VNI": 11
}
}
kind: ConfigMap
kind: ConfigMap
metadata:
annotations:
labels:
app: flannel
tier: node
name: kube-flannel-cfg
namespace: kube-system
resourceVersion: "79463"
uid: 0d3daba1-4db8-44b7-80e0-c73965977035


2. Modify in all nodes (kubectl edit node ) with VNI 11 from VNI 1 including BIGIP node.
3. Try restarting the flannel pods in kube-system. If doesn’t work.
4. Reboot the cluster. This should bringup the cluster with necessary VNI and network change.
```