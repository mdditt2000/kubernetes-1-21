# Multi-Cluster Kubernetes using Container Ingress Services and BIG-IP

Today, organizations are increasingly deploying many more Kubernetes clusters. Deploying multiple Kubernetes clusters can improve availability, isolation and scalability. This user-guide documents two Kubernetes clusters using Container Ingress Services (CIS) to automates BIP-IP to provide Edge Ingress services.   

## Multi-Cluster Application Architecture

In this user-guide, each cluster runs a full copy of the application. This simple but powerful approach enables an application to be graduated from dev to prod. Future user-guides will focus on multiple availability zones or data centers and user traffic routed to the closest or most appropriate cluster using health-aware global load balancer. 

### Environment parameters

* Multiple K8S 1.21 custer - one master and two worker nodes
* CIS 2.4.0
* AS3: 3.26
* BIG-IP 15.1

## Create Vxlan tunnels and self-IPs for the cluster

* tmsh create net tunnels vxlan fl-vxlan port 8472 flooding-type none
* tmsh create net tunnel tunnel vxlan-tunnel-dev key 1 profile fl-vxlan local-address 192.168.200.60
* tmsh create net tunnel tunnel vxlan-tunnel-prod key 11 profile fl-vxlan local-address 192.168.200.60
* tmsh create net self 10.244.20.60 address 10.244.20.60/255.255.0.0 allow-service none vlan vxlan-tunnel-dev
* tmsh create net self 10.245.20.60 address 10.245.20.60/255.255.0.0 allow-service none vlan vxlan-tunnel-prod

## Example of Vxlan tunnels and self-IPs for the cluster

Tunnel profile fl-vxlan configuration for vxlan-tunnel-dev and vxlan-tunnel-dev

![vxlan-tunnel](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-04_10-24-52.png)

Self-IPs configuration for vxlan-tunnel-dev and vxlan-tunnel-dev

![self-ip](https://github.com/mdditt2000/kubernetes-1-21/blob/main/cis%202.4/multi-cluster/diagrams/2021-05-04_10-30-14.png)


Tunnel profile fl-vxlan configuration for vxlan-tunnel-dev

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