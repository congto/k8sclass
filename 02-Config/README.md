# 2. Configure Kubernetes

The previous lab just did all the set up for you but there is still some steps we need to complete!

## kubectl

We need to figure out what our machine nginx01 load balancer is so we can log into the machine

```
openstack server list | grep <your nginx host name>
```

You will see two IP addresses.  The public IP address is the one we are after.  Make note of this IP.  It might be easiest to set an environment variable: 

```
export PUBLIC_IP=<nginx public IP>
```

Now we need to set up ```kubectl``` so it can communicate with the cluster. This section is from [Kelsey Hightower's Kuberentes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/06-kubectl.md)

Name your cluster something fun and unique:

```
export CLUSTER=<cluster_name>
```
Now let's configure ```kubectl```

```
cd <certs directory>
kubectl config set-cluster $CLUSTER \
	--certificate-authority=ca.pem \
	--embed-certs=true --server=https://$PUBLIC_IP
```

Remember the token we set in the Terraform file?  You can fish that out and run the following command so we can speak with that cluster: 

```
kubectl config set-credentials admin --token <token>
```

```
kubectl config set-context default-context \
  --cluster=$CLUSTER \
  --user=admin
```

You should be able to connect from your local machine to the kubernetes cluster through the nginx load balancer:

```
kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}
```
```
kubectl get nodes
NAME        STATUS    AGE
kworker01   Ready     1h
kworker02   Ready     1h
kworker03   Ready     1h
```

[More source info](http://kubernetes.io/docs/user-guide/kubectl-cheatsheet/)

## Configure overlay networking

In most of the kubernetes guides you'll find recommendations to use weave, flannel, calico, or some other networking overlay.  With OpenStack, we already have a network overlay called neutron, so we'll use it!

(Note: For more information on this section see [this excellent blog post](http://blogs.cisco.com/cloud/deploy-a-kubernetes-cluster-on-openstack-using-ansible))

We'll have to first define our static routes.  Run the following command to figure out which node is responsible for which network: 

```
kubectl get nodes  --output=jsonpath='{range .items[*]}{.status.addresses[?(@.type=="InternalIP")].address} {.spec.podCIDR} {"\n"}{end}' | awk '{print $2 "," $1}'
```
This gives us output like: 

```
10.200.2.0/24,10.106.1.26
10.200.1.0/24,10.106.1.28
10.200.0.0/24,10.106.1.30
```
Which is exactly what we can put into our subnets.

__IMPORTANT__ you will need openstackclient 3.3 for this command to work: 

```
pip install --upgrade python-openstackclient
```
Next we find all the ports in our network: 

```
openstack port list --network pipeline
```


We will use neutron as the overlay network instead of something like weave or flannel.  This is done on an admin account with a command like: 

```
neutron port-update 1b6b5e18-bd6c-4924-804f-bfa61432d4b4 \
--allowed-address-pairs type=dict list=true \
ip_address=10.200.2.0/24 \
ip_address=192.168.0.0/16 \
ip_address=172.31.232.0/21
```
Here this is the UUID of the port and then we are saying which subnets we allow through the port. 

```
neutron port-update ce036350-e199-423a-a1cf-480f009f9f91 \
--allowed-address-pairs type=dict list=true ip_address=10.201.1.0/24
```
```
neutron port-update adbce357-da18-4b2c-aef7-3c17c67bbc43 \
--allowed-address-pairs type=dict list=true ip_address=10.201.2.0/24
```
Check that its working with: 

```
neutron port-show c49e2845-d20b-41da-b18a-30c4ada8e97a
+-----------------------+-------------------------------------------------------------------------------------+
| Field                 | Value                                                                               |
+-----------------------+-------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                |
| allowed_address_pairs | {"ip_address": "10.201.0.0/24", "mac_address": "fa:16:3e:16:d0:44"}                 |
| binding:host_id       | mhv6.trial5.mc.metacloud.in                                                         |
| binding:profile       | {}                                                                                  |
| binding:vif_details   | {"port_filter": true}                                                               |
| binding:vif_type      | bridge                                                                              |
| binding:vnic_type     | normal                                                                              |
| device_id             | 49cc915e-e04f-47da-a234-9d29cbb5cce5                                                |
| device_owner          | compute:None                                                                        |
| extra_dhcp_opts       |                                                                                     |
| fixed_ips             | {"subnet_id": "e9e30169-44b0-4b17-8aaa-77852190875e", "ip_address": "10.106.1.144"} |
| id                    | c49e2845-d20b-41da-b18a-30c4ada8e97a                                                |
| mac_address           | fa:16:3e:16:d0:44                                                                   |
| name                  |                                                                                     |
| network_id            | b0c2ce4c-d706-4094-b7fb-243ddeada563                                                |
| security_groups       | 289dd19a-194d-4026-915b-d75ec4d890c1                                                |
| status                | ACTIVE                                                                              |
| tenant_id             | 0b42f5efc6de46cb8a3119e5f667b868                                                    |
+-----------------------+-------------------------------------------------------------------------------------+
```
Notice that we see the port 

__IMPORTANT:__ At this point give the instructor the output of this last command.  When the instructor gives you the green light, reboot your nodes for these routes to take effect. 
 
Upon reboot you can go into the nginx server and verify that the routes are set with

```
route -n 
```
You should see the static routes configured in the route table: 

```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.106.0.1      0.0.0.0         UG    0      0        0 ens3
10.106.0.0      0.0.0.0         255.255.0.0     U     0      0        0 ens3
10.200.0.0      10.106.1.30     255.255.255.0   UG    0      0        0 ens3
10.200.1.0      10.106.1.28     255.255.255.0   UG    0      0        0 ens3
10.200.2.0      10.106.1.26     255.255.255.0   UG    0      0        0 ens3
169.254.169.254 10.106.0.2      255.255.255.255 UGH   0      0        0 ens3
```

## Kubernetes DNS

```
kubectl create -f https://raw.githubusercontent.com/kelseyhightower/kubernetes-the-hard-way/master/services/kubedns.yaml
```

```
kubectl --namespace=kube-system get svc
```

```
kubectl create -f https://raw.githubusercontent.com/kelseyhightower/kubernetes-the-hard-way/master/deployments/kubedns.yaml
```

Check that services are up

```
kubectl --namespace=kube-system get deployments
kubectl --namespace=kube-system get svc
kubectl --namespace=kube-system get pods
```
Check individual nodes to see if they are up: 

```
