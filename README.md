# metallb-testing for SCTP trafic
playing around with metallb and testing with SCTP traffic 

## Lab Description and Installation:

### VM Setup 
The setup is a based on a single cluster VM deployed with multipass:
```
multipass launch --name test-metalbv2 --mem 8G --disk 10G --cpus 4 focal
```

       test-metalbv2                fiveg-host-24-node4
       +---------+                  +----------------+
       |  app    |                  |                |
       | cluster +------------------+ client (host)  |
       |         |                  |                |
       +---------+                  +----------------+
        .106 |                         | .1
             +--------- Network -------+
                    10.65.94.0/24
                    
Make sure to set up ssh access for conviniency.

### K3s installation

Login to the cluster VM and install K3s with "--disable servicelb" option *otherwise it won't work*

```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --disable servicelb" sh -
```
Other consideratiosn for Installation (nothing fancy here)
* copy kube credentials and change permissions if you're lazy and don't want "sudo k3s kubectl" -
* kube completion

## Test metalb with standard NGINX deployment

### start nginx deployment with service

First let's do a basic deployment in a VM by deploying 3 pods and a clusterIP service (default service type).
```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
EOF
```

All good, service is created and working

```console
ubuntu@test-metalbv2:~$ kubectl get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.43.0.1      <none>        443/TCP   29m
nginx-service   ClusterIP   10.43.174.62   <none>        80/TCP    15m
ubuntu@test-metalbv2:~$ 
ubuntu@test-metalbv2:~$ curl http://10.43.174.62/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### Add metalb with IP on VNIC 

*Deploy metallb*

We'll use info from https://metallb.universe.tf/installation/
We also choose the k8s/FRR version because we're a bunch of network nerds.
``` 
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-frr-k8s.yaml
```
*Configure Metalb and TCP service (nginx deployment)*

We deploy now a load balancer service exposing an external IP set to 10.65.94.100 and we test the reachability of the nginx service.
```
                  +-----------------------------------------------------+  
                  |                                                     |  
                  |  +----------------------------------------------+   |  
                  |  |                                              |   |  
                  |  | +------------+ +------------+ +------------+ |   | 
                  |  | | nginx pod1 | | nginx pod2 | | nginx pod3 | |   |  
                  |  | +------------+ +------------+ +------------+ |   | 
                  |  |                 10.42/16                     |   |  
                  |  +----------------------------------------------+   |  
                  |                      deployment                     | 
                  |                                                     |  
                  |                                                     |  
                  +-------------------+-   -+---------------------------+  
                                      |     | NIC                          
                                      |     |                              
                                      |     | IP = 10.65.94.1/24           
                                      +--|--+ LOAD BALANCER = 10.65.94.100  
                                         |             
                                         |                                    
                      -------------------|----------------              
                                            ^                           
                                            |                               
                                            |                                 
                     curl http://10.65.94.100:80/ from Host (client)         
```
Now let's deploy:
- Load balancer service (with metalb annotion)
- metalb service with pool configuration and L2 mode
  
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  annotations:
    metallb.universe.tf/address-pool: mypool
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
  loadBalancerIP: 10.65.94.100
```
```
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: mypool
  namespace: metallb-system
spec:
  addresses:
  - 10.65.94.0/24
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
EOF
```

```console
root@fiveg-host-24-node4:~# curl http://10.65.94.100:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```


### Under the hood. This all iptables (business as usual)
```
###################### There is no new IP Address configured on the interface 
 
ubuntu@test-metalbv2:~$ ip addr show | grep 10.65
    inet 10.65.94.106/24 brd 10.65.94.255 scope global ens3
ubuntu@test-metalbv2:~$

###################### this is all iptables (is proxy-arp enforced ?)
ubuntu@test-metalbv2:~$ sudo iptables-save 
[...]
-A KUBE-SERVICES -d 10.65.94.100/32 -p tcp -m comment --comment "default/nginx-service loadbalancer IP" -m tcp --dport 80 -j KUBE-EXT-V2OKYYMBY3REGZOG
-A KUBE-EXT-V2OKYYMBY3REGZOG -j KUBE-SVC-V2OKYYMBY3REGZOG
-A KUBE-SVC-V2OKYYMBY3REGZOG ! -s 10.42.0.0/16 -d 10.43.67.168/32 -p tcp -m comment --comment "default/nginx-service cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-V2OKYYMBY3REGZOG -m comment --comment "default/nginx-service -> 10.42.0.10:80" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-BUR35ICIEJ6FUKE3
-A KUBE-SVC-V2OKYYMBY3REGZOG -m comment --comment "default/nginx-service -> 10.42.0.8:80" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-LNMZPQ2U2A5TEEGP
-A KUBE-SVC-V2OKYYMBY3REGZOG -m comment --comment "default/nginx-service -> 10.42.0.9:80" -j KUBE-SEP-F6NN6HBKB5TWKYLC
[...]
```

This worked! 


## SCTP integration

### Basic SCTP install and check

Install lksctp-tools in *BOTH* VM and host. This will install kernel modules too and permits to test that environment is capable of processing SCTP.

```
sudo apt install lksctp-tools -y && sudo apt install telnet -y
```
Start server from VM and test SCTP.
```console
ubuntu@test-metalbv2:~$ sctp_test -H 10.65.94.106 -P 4444 -l
local:addr=10.65.94.106, port=4444, family=2
seed = 1716370976

Starting tests...
        socket(SOCK_SEQPACKET, IPPROTO_SCTP)  ->  sk=3
        bind(sk=3, [a:10.65.94.106,p:4444])  --  attempt 1/10
        listen(sk=3,backlog=100)
Server: Receiving packets.
        recvmsg(sk=3) Notification: SCTP_ASSOC_CHANGE(COMMUNICATION_UP)
                (assoc_change: state=0, error=0, instr=10 outstr=10)
        recvmsg(sk=3) Data 2 bytes. First 2 bytes: .

        recvmsg(sk=3) Data 2 bytes. First 2 bytes: .

          SNDRCV(stream=0 ssn=1 tsn=2145695259 flags=0x0 ppid=0
cumtsn=0
        recvmsg(sk=3) Data 2 bytes. First 2 bytes: .
```
``` I
root@fiveg-host-24-node4:~# withsctp telnet  10.65.94.106 4444
Trying 10.65.94.106...
Connected to 10.65.94.106.
Escape character is '^]'.
```

### test with  sctp server deployment and metallb loadbalancer

```
                  +-----------------------------------------------------+  
                  |                                                     |  
                  |  +----------------------------------------------+   |  
                  |  |                                              |   |  
                  |  | +------------+ +------------+ +------------+ |   | 
                  |  | | sctp-server| | sctp-server| | sctp-server| |   |  
                  |  | +------------+ +------------+ +------------+ |   | 
                  |  |                 10.42/16                     |   |  
                  |  +----------------------------------------------+   |  
                  |                      deployment                     | 
                  |                                                     |  
                  | LOAD BALANCER for SCTP = 10.65.94.101 / PORT 9999   |  
                  +-------------------+-   -+---------------------------+  
                                      |     | NIC                          
                                      |     |                              
                                      |     | IP = 10.65.94.1/24           
                                      +--|--+ 
                                         |             
                                         |                                    
                      -------------------|----------------              
                                            ^                           
                                            |                               
                                            |                                 
                      from Host (client): sctp_test -H 10.65.94.1 -h  10.65.94.101 -p 9999 -s         
``` 

Configure a deployment with replica = 1 first. 

```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sctp-server
  labels:
    app: sctp-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sctp-server
  template:
    metadata:
      labels:
        app: sctp-server
    spec:
      containers:
      - name: sctp-server
        image: ubuntu:20.04
        command: ["/bin/bash", "-c", "apt-get update && apt-get install -y lksctp-tools && echo 'Starting SCTP server on port 9999' && sctp_test -H 0.0.0.0 -P 9999 -l"]
        ports:
        - containerPort: 9999
          protocol: SCTP
---
apiVersion: v1
kind: Service
metadata:
  name: sctp-server
  annotations:
    metallb.universe.tf/address-pool: mypool
spec:
  selector:
    app: sctp-server
  ports:
  - protocol: SCTP
    port: 9999
    targetPort: 9999
  type: LoadBalancer
  loadBalancerIP: 10.65.94.101
```

This just worked !

```console
###
### FROM HOST (client):
###
root@fiveg-host-24-node4:~# sctp_test -H 10.65.94.1 -h  10.65.94.101 -p 9999 -s

###
### FROM Cluster VM:
###
ubuntu@test-metalbv2:~$ kubectl logs      sctp-server-84f9bffb47-hfln5    -f
[....]
Starting tests...
        socket(SOCK_SEQPACKET, IPPROTO_SCTP)  ->  sk=3
        bind(sk=3, [a:0.0.0.0,p:9999])  --  attempt 1/10
        listen(sk=3,backlog=100)
Server: Receiving packets.
        recvmsg(sk=3) Notification: SCTP_ASSOC_CHANGE(COMMUNICATION_UP)
                (assoc_change: state=0, error=0, instr=10 outstr=10)
        recvmsg(sk=3) Data 1 bytes. First 1 bytes: <empty> text[0]=0
        recvmsg(sk=3) Data 1 bytes. First 1 bytes: <empty> text[0]=0
          SNDRCV(stream=0 ssn=0 tsn=50276345 flags=0x1 ppid=641842057
cumtsn=50276345
        recvmsg(sk=3) Data 1 bytes. First 1 bytes: <empty> text[0]=0
          SNDRCV(stream=0 ssn=0 tsn=50276419 flags=0x1 ppid=611453669
cumtsn=50276345
        recvmsg(sk=3) Data 1 bytes. First 1 bytes: <empty> text[0]=0
          SNDRCV(stream=0 ssn=0 tsn=50276420 flags=0x1 ppid=493756153
cumtsn=50276345
        recvmsg(sk=3) Data 1 bytes. First 1 bytes: <empty> text[0]=0
          SNDRCV(stream=0 ssn=0 tsn=50276421 flags=0x1 ppid=217965793
cumtsn=50276345
        recvmsg(sk=3) Data 1 bytes. First 1 bytes: <empty> text[0]=0
          SNDRCV(stream=0 ssn=0 tsn=50276422 flags=0x1 ppid=2114115591

```






