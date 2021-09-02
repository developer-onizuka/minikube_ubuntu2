# 0. server's functions

| deployment | VM | ExposedIP | clusterIP | container's port |
| --- | --- | --- | --- | --- |
| nginx-test | minikube | 192.168.99.109 | don't care. resolved by dns | 80 | 
| mongo-test | minikube-m03 | N/A | don't care. resolved by dns | 27017 | 
| employee-test | minikube-m02 | N/A | don't care. resolved by dns | 5001 |


# 1. Install each package on Ubuntu20.4
```
<minikube>
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
$ sudo dpkg -i minikube_latest_amd64.deb

<kubectl>
$ sudo apt-get update && sudo apt-get install -y apt-transport-https gnupg2
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
$ sudo apt-get update
$ sudo apt-get install -y kubectl
```
# 2. Run minikube
```
$ minikube start --driver=virtualbox --nodes=3
😄  minikube v1.19.0 on Ubuntu 20.04
✨  Using the virtualbox driver based on user configuration
💿  Downloading VM boot image ...
 ...
🔎  Verifying Kubernetes components...
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

# 3. Check the node list and status
```
$ minikube node list
minikube	192.168.99.109
minikube-m02	192.168.99.110
minikube-m03	192.168.99.111

$ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

minikube-m02
type: Worker
host: Running
kubelet: Running

minikube-m03
type: Worker
host: Running
kubelet: Running
```

# 3. Put labels on each node
```
$ kubectl label nodes minikube location=tokyo
node/minikube labeled
$ kubectl label nodes minikube-m02 location=telaviv
node/minikube-m02 labeled
$ kubectl label nodes minikube-m03 location=houston
node/minikube-m03 labeled
$ kubectl describe node |grep location
                    location=tokyo
                    location=telaviv
                    location=houston
```

# 4. Make mongo's yaml file and run it. Check where the pod is running.
```
$ kubectl create -f mongo-pv.yaml 
$ kubectl create -f mongo-pvc.yaml

$ kubectl create -f mongo_latest_nodeSelector_svc.yaml 
deployment.apps/mongo-test created

$ kubectl describe pod mongo-test |grep Node:
Node:         minikube-m03/192.168.99.111
```
# 5. Check if mongo's deployment is exposed.
```
$ minikube service list
|-------------|------------|--------------|-----|
|  NAMESPACE  |    NAME    | TARGET PORT  | URL |
|-------------|------------|--------------|-----|
| default     | kubernetes | No node port |
| default     | mongo-test | No node port |
| kube-system | kube-dns   | No node port |
|-------------|------------|--------------|-----|

$ kubectl create -f dnsutils_latest.yaml 
pod/dnsutils created

$ kubectl exec -it dnsutils -- /bin/sh
# nslookup mongo-tset
Server:		10.96.0.10
Address:	10.96.0.10#53

** server can't find mongo-tset: NXDOMAIN

# nslookup mongo-test
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	mongo-test.default.svc.cluster.local
Address: 10.105.128.151

# curl mongo-test:27017
It looks like you are trying to access MongoDB over HTTP on the native driver port.

```

# 6. Make employee's yaml file and run it. Check where the pod is running.
```
$ kubectl create -f employee_latest_nodeSelector_svc.yaml

$ kubectl describe pod employee-test |grep Node:
Node:         minikube-m02/192.168.99.110
```

# 7. Check if employee's deployment is exposed.
```
$ minikube service list
|-------------|---------------|--------------|-----|
|  NAMESPACE  |     NAME      | TARGET PORT  | URL |
|-------------|---------------|--------------|-----|
| default     | employee-test | No node port |
| default     | kubernetes    | No node port |
| default     | mongo-test    | No node port |
| kube-system | kube-dns      | No node port |
|-------------|---------------|--------------|-----|
```

# 8. Check if it works correcly 
```
$ kubectl -it exec dnsutils -- /bin/sh
# curl https://employee-test:5001 -k
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title> - Employee</title>
    <link rel="stylesheet" href="/lib/bootstrap/dist/css/bootstrap.min.css" />
    <link rel="stylesheet" href="/css/site.css" />
</head>
 ...
    <script src="/lib/jquery/dist/jquery.min.js"></script>
    <script src="/lib/bootstrap/dist/js/bootstrap.bundle.min.js"></script>
    <script src="/js/site.js"></script>
    
</body>
</html>
```
# 9. Create the persistent volume and make nginx's config on it.
```     
$ kubectl create -f nginx-pv.yaml 
$ kubectl create -f nginx-pvc.yaml 

$ ssh docker@192.168.99.109
docker@192.168.99.109's password: tcuser

$ sudo mkdir -p /var/data
$ sudo vi /var/data/default.conf 
upstream proxy.com {
        server employee-test:5001;
}

server {
        listen 80;
        server_name localhost;
        location / {
                root /usr/share/nginx/html;
                index index.html index.htm;
                proxy_pass https://proxy.com;
        }
}

```
# 10. Make nginx's yaml file and run it. 
```
$ kubectl create -f nginx_1.14.2_nodeSelector_svc.yaml 
deployment.apps/nginx-test created

$ kubectl describe pod nginx-test |grep Node:
Node:         minikube/192.168.99.109
```

# 11. Check if employee's deployment is exposed. Access the following URL from Host's blowser.
```
$ minikube service list
|-------------|---------------|--------------|-----------------------------|
|  NAMESPACE  |     NAME      | TARGET PORT  |             URL             |
|-------------|---------------|--------------|-----------------------------|
| default     | employee-test | No node port |
| default     | kubernetes    | No node port |
| default     | mongo-test    | No node port |
| default     | nginx-test    | http/8080    | http://192.168.99.109:32502 |
| kube-system | kube-dns      | No node port |
|-------------|---------------|--------------|-----------------------------|

```
