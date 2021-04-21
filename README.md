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
$ cat mongo_latest_nodeSelector_svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: mongo-test
  labels:
    run: mongo-test
spec:
#  type: NodePort
  ports:
  - port: 27017
    targetPort: 27017 
    protocol: TCP
    name: mongo
  selector:
    run: mongo-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-test
spec:
  selector:
    matchLabels:
      run: mongo-test
  replicas: 1
  template:
    metadata:
      labels:
        run: mongo-test
    spec:
      containers:
      - name: mongodb
        image: docker.io/mongo
        ports:
        - containerPort: 27017
      nodeSelector:
        location: houston

$ kubectl create -f mongo_latest_nodeSelector_svc.yaml 
deployment.apps/mongo-test created

$ kubectl describe pod mongo-test
Name:         mongo-test-7cb544b5bd-nm7dk
Namespace:    default
Priority:     0
Node:         minikube-m03/192.168.99.111
Start Time:   Wed, 21 Apr 2021 12:51:13 +0900
Labels:       pod-template-hash=7cb544b5bd
              run=mongo-test
Annotations:  <none>
Status:       Running
IP:           10.244.2.8
IPs:
  IP:           10.244.2.8
Controlled By:  ReplicaSet/mongo-test-7cb544b5bd
Containers:
  mongodb:
    Container ID:   docker://655d4c58396e5985af52058fff729888dd90982ec5a6813ca108d3a75ccb6ce1
    Image:          docker.io/mongo
    Image ID:       docker-pullable://mongo@sha256:b66f48968d757262e5c29979e6aa3af944d4ef166314146e1b3a788f0d191ac3
    Port:           27017/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 21 Apr 2021 12:51:18 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-9hsv7 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-9hsv7:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-9hsv7
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  location=houston
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  90s   default-scheduler  Successfully assigned default/mongo-test-7cb544b5bd-nm7dk to minikube-m03
  Normal  Pulling    89s   kubelet            Pulling image "docker.io/mongo"
  Normal  Pulled     85s   kubelet            Successfully pulled image "docker.io/mongo" in 3.750146474s
  Normal  Created    85s   kubelet            Created container mongodb
  Normal  Started    85s   kubelet            Started container mongodb
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

$ cat dnsutils_latest.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  labels:
    name: dnsutils
spec:
  containers:
  - name: dnsutils
    image: tutum/dnsutils
    command:
    - sleep
    - "3600"

$ kubectl create -f dnsutils_latest.yaml 
pod/dnsutils created

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
$ cat employee_latest_nodeSelector_svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: employee-test
  labels:
    run: employee-test
spec:
#  type: NodePort
  ports:
  - port: 5001
    targetPort: 5001
    protocol: TCP
    name: https
  - port: 5000
    targetPort: 5000
    protocol: TCP
    name: http
  selector:
    run: employee-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: employee-test
spec:
  selector:
    matchLabels:
      run: employee-test
  replicas: 1
  template:
    metadata:
      labels:
        run: employee-test
    spec:
      containers:
      - name: employee
        image: developeronizuka/employee
        ports:
        - containerPort: 5001
        - containerPort: 5000
        env:
        - name: MONGO
          value: mongo-test
        command: ["/usr/local/dotnet/publish/Employee"]
      nodeSelector:
        location: telaviv

$ kubectl describe pod employee-test
Name:         employee-test-84b567445f-fw8k7
Namespace:    default
Priority:     0
Node:         minikube-m02/192.168.99.110
Start Time:   Wed, 21 Apr 2021 00:01:33 +0900
Labels:       pod-template-hash=84b567445f
              run=employee-test
Annotations:  <none>
Status:       Running
IP:           10.244.1.9
IPs:
  IP:           10.244.1.9
Controlled By:  ReplicaSet/employee-test-84b567445f
Containers:
  employee:
    Container ID:  docker://5d33b42a535e512120e1194a404c6447c5a1c684bccb6fb55390abdc6d4630eb
    Image:         developeronizuka/employee
    Image ID:      docker-pullable://developeronizuka/employee@sha256:ad36f06fcb5aa8d4da7dc36ac9bf42223617c3330e8dfcaef1b5a30ba9f71084
    Ports:         5001/TCP, 5000/TCP
    Host Ports:    0/TCP, 0/TCP
    Command:
      /usr/local/dotnet/publish/Employee
    State:          Running
      Started:      Wed, 21 Apr 2021 00:01:36 +0900
    Ready:          True
    Restart Count:  0
    Environment:
      MONGO:  mongo-test
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-9hsv7 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-9hsv7:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-9hsv7
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  location=telaviv
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  4m19s  default-scheduler  Successfully assigned default/employee-test-84b567445f-fw8k7 to minikube-m02
  Normal  Pulling    4m18s  kubelet            Pulling image "developeronizuka/employee"
  Normal  Pulled     4m16s  kubelet            Successfully pulled image "developeronizuka/employee" in 2.416408504s
  Normal  Created    4m16s  kubelet            Created container employee
  Normal  Started    4m16s  kubelet            Started container employee


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
<body>
    <header>
        <nav class="navbar navbar-expand-sm navbar-toggleable-sm navbar-light bg-white border-bottom box-shadow mb-3">
            <div class="container">
                <a class="navbar-brand" href="/">Employee</a>
                <button class="navbar-toggler" type="button" data-toggle="collapse" data-target=".navbar-collapse" aria-controls="navbarSupportedContent"
                        aria-expanded="false" aria-label="Toggle navigation">
                    <span class="navbar-toggler-icon"></span>
                </button>
                <div class="navbar-collapse collapse d-sm-inline-flex flex-sm-row-reverse">
                    <ul class="navbar-nav flex-grow-1">
                        <li class="nav-item">
                            <a class="nav-link text-dark" href="/">Home</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link text-dark" href="/Home/Privacy">Privacy</a>
                        </li>
                    </ul>
                </div>
            </div>
        </nav>
    </header>
    <div class="container">
        <main role="main" class="pb-3">
            
<h1>List of Employees</h1>

<h2></h2>

<a href="/Home/Insert"> Add New Employee</a>

<br /><br />


<table border="1" cellpadding="10">
</table>

<div class="text-center">
    <h1 class="display-4">Welcome</h1>
    <p>Learn about <a href="https://docs.microsoft.com/aspnet/core">building Web apps with ASP.NET Core</a>.</p>
</div>

        </main>
    </div>

    <footer class="border-top footer text-muted">
        <div class="container">
            &copy; 2020 - Employee - <a href="/Home/Privacy">Privacy</a>
        </div>
    </footer>
    <script src="/lib/jquery/dist/jquery.min.js"></script>
    <script src="/lib/bootstrap/dist/js/bootstrap.bundle.min.js"></script>
    <script src="/js/site.js"></script>
    
</body>
</html>
```
# 9. Create the persistent volume and make nginx's config on it.
```
$ cat nginx-pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv
spec:
  storageClassName: standard
  volumeMode: Filesystem
  capacity:
    storage: 1Gi
  persistentVolumeReclaimPolicy: Retain
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/var/data"
    type: DirectoryOrCreate

$ cat nginx-pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hostpath-pvc
spec:
  storageClassName: standard
  accessModes:
  - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi

$ ssh docker@192.168.99.109
docker@192.168.99.109's password: tcuser

$ cat /var/data/default.conf 
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
$ cat nginx_1.14.2_nodeSelector_svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
  labels:
    run: nginx-test
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    run: nginx-test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  selector:
    matchLabels:
      run: nginx-test
  replicas: 1
  template:
    metadata:
      labels:
        run: nginx-test
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-conf
        persistentVolumeClaim:
         claimName: hostpath-pvc
      nodeSelector:
        location: tokyo

$ kubectl create -f nginx_1.14.2_nodeSelector_svc.yaml 
deployment.apps/nginx-test created

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
