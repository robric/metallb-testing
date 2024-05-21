# metallb-testing
playing around with metallb and testing with SCTP traffic 

## Lab Description and Installation:

The setup is a based on two VMs deployed with multipass:

       +---------+                  +---------+
       |  app    |                  |         |
       | cluster +------------------+ client  |
       |         |                  |         |
       +---------+                  +---------+
        .154 |                         | .94
             +--------- Network -------+
                    10.65.94.0.24
Installation - copy kube credentials and change permissions if you're lazy and don't want "sudo k3s kubectl" -
```
multipass launch --name test-metalb --mem 32G --disk 200G --cpus 8 focal
multipass launch --name client --mem 2G --disk 5G --cpus 2 focal
```
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

All good, service is created

``
ubuntu@test-metalb:~$ kubectl get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.43.0.1      <none>        443/TCP   29m
nginx-service   ClusterIP   10.43.174.62   <none>        80/TCP    15m
ubuntu@test-metalb:~$ 
ubuntu@test-metalb:~$ curl http://10.43.174.62/
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

## 



