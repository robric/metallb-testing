# metallb-testing
playing around with metallb and testing with SCTP traffic 

## Lab Description and Installation:

### VM Setup 
The setup is a based on a single cluster VM deployed with multipass:
```
multipass launch --name test-metalbv2 --mem 8G --disk 10G --cpus 4 focal
```

       test-metalbv2
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

## Service deployment NGINX

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

## Now let's add a metalb LB on the main interface

### Deploy metallb

We'll use info from https://metallb.universe.tf/installation/
We also choose the k8s/FRR version because we're a bunch of network nerds.
``` 
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-frr-k8s.yaml
```
### Configure Metalb and Services

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

This worked:
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




