### CNLB-Lab : Create quick testbed to test k8s load-balancing implementations. 

### Pre-requisites
- Server/PC with Linux (> 16 cores, > 32Gb RAM)
- VirtualBox installed
- Vagrant installed

### Supported LBs 
 - LoxiLB
 - MetalLB

### How to setup ??

```
git clone https://github.com/codeSnip12/cnlblab
cd cnlblab
vagrant up
```

The following setup would be created: 
```
+--------------+        +---------------+
|    master    |        |   worker1     |
| 192.168.80.10|        | 192.168.80.101|
+------+-------+        +-------+-------+
       |                        |
       |                        |
       +-----------+------------+
                   |
                   |
             +-----+------+
             |   host     |
             |192.168.80.9|
             +------------+

```


### How to setup and test LoxiLB ??

#### Login into master node 
```
vagrant ssh master
```
#### Install LoxiLB and iperf pods
```
sudo kubectl apply -f /vagrant/manifests/kube-loxilb.yml 
sudo kubectl apply -f /vagrant/manifests/loxilb.yml
sudo kubectl apply -f /vagrant/manifests/iperf-service-loxilb.yml
```
#### Check everything is up and running 
```
vagrant@master:~$ sudo kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   local-path-provisioner-84db5d44d9-65kcg   1/1     Running   0          12m
kube-system   coredns-6799fbcd5-c5bs9                   1/1     Running   0          12m
kube-system   metrics-server-67c658944b-dtjkf           1/1     Running   0          12m
kube-system   kube-loxilb-745fb47486-nwmp8              1/1     Running   0          3m8s
kube-system   loxilb-lb-vkpm9                           1/1     Running   0          2m28s
kube-system   loxilb-lb-thpx8                           1/1     Running   0          2m28s
```
#### Run test

```
vagrant ssh host
$ iperf -c 192.168.80.20 -p 55001 -i1 -t10
```
#### Cleanup LoxiLB (if need to run MetalLB next) from master node
```
sudo kubectl delete -f /vagrant/manifests/iperf-service-loxilb.yml 
sudo kubectl delete -f /vagrant/manifests/loxilb.yml 
sudo kubectl delete -f /vagrant/manifests/kube-loxilb.yml 
```
### How to setup and test MetalLB ??

#### Setup master node (MetalLB needs strictARP IPVS mode)
```
vagrant ssh master
$ sudo su 
$ echo 1 > /proc/sys/net/ipv4/conf/eth1/arp_ignore
$ echo 2 > /proc/sys/net/ipv4/conf/eth1/arp_announce
```
#### Setup worker node (MetalLB needs strictARP IPVS mode)
```
vagrant ssh worker1
$ sudo su 
$ echo 1 > /proc/sys/net/ipv4/conf/eth1/arp_ignore
$ echo 2 > /proc/sys/net/ipv4/conf/eth1/arp_announce
```

#### Install MetalLB and iperf pods from master node
```
sudo kubectl apply -f /vagrant/manifests/metallb-native.yaml
sudo kubectl apply -f /vagrant/manifests/metallb-addr-pool.yml
sudo kubectl apply -f /vagrant/manifests/iperf-service-metallb.yml 
```
#### Check everything is up and running 
```
vagrant@master:~$ sudo kubectl get pods -A
NAMESPACE        NAME                                      READY   STATUS    RESTARTS   AGE
kube-system      local-path-provisioner-84db5d44d9-65kcg   1/1     Running   0          22m
kube-system      coredns-6799fbcd5-c5bs9                   1/1     Running   0          22m
kube-system      metrics-server-67c658944b-dtjkf           1/1     Running   0          22m
metallb-system   controller-786f9df989-pv2qv               1/1     Running   0          89s
metallb-system   speaker-lmr7r                             1/1     Running   0          89s
metallb-system   speaker-9nksg                             1/1     Running   0          89s
```
#### Run test

```
vagrant ssh host
$ iperf -c 192.168.80.20 -p 55001 -i1 -t10
```

### How to move VIP assignment from one to another node
- For loxilb, check if VIP is assigned in lo interface of master/worker and delete pod loxilb-lb-xxxx  running in that node.
- For metallb, could not find exact method. So, added static route for VIP (192.168.80.20 via 192.168.80.10 or 192.168.80.20 via 192.168.80.101) at host


