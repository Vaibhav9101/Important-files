Master & Worker Node
Step1:

sudo apt update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo apt install docker.io -y

sudo systemctl enable --now docker # enable and start in single command.

# Adding GPG keys.
curl -fsSL "https://packages.cloud.google.com/apt/doc/apt-key.gpg" | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/kubernetes-archive-keyring.gpg

# Add the repository to the sourcelist.
echo 'deb https://packages.cloud.google.com/apt kubernetes-xenial main' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update 
sudo apt install kubeadm=1.20.0-00 kubectl=1.20.0-00 kubelet=1.20.0-00 -y

Note: 
Kubeadm: Kubeadm is a tool to built Kubernetes clusters. It’s responsible for cluster bootstrapping.Kubeadm runs a series of prechecks to ensure that the machine is ready to run Kubernetes, during bootstraping the cluster kubeadm is downloading and install the cluster control plane components and configure all necessary cluster resources.

kubectl: the command line util to talk to your cluster

kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
----------------------------------------------------------------------------------------------------------------------------------------------------
Master Node
Step2:
sudo kubeadm init

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

sudo kubeadm token create --print-join-command   (Optional)
----------------------------------------------------------------------------------------------------------------------------------------------------
Worker Node
sudo kubeadm reset pre-flight checks

sudo kubeadm join 172.31.35.249:6443 --token 1xe2pz.2623jfsgp2f8he8l \
    --discovery-token-ca-cert-hash sha256:cf717b25825d87164e3d1346a2dda2722c4bf1afaf642518727a8277a556549c --v=5
-----------------------------------------------------------------------------------------------------------------------------------------------------
Optional: Labeling Nodes
If you want to label worker nodes, you can use the following command:

kubectl label node <node-name> node-role.kubernetes.io/worker=worker
------------------------------------------------------------------------------------------------------------------------------------------------------
kubectl get nodes
------------------------------------------------------------------------------------------------------------------------------------------------------

vi pod.yaml
------------
apiVersion: v1
kind: Pod
metadata:
  name: two-tier-app-pod
spec:
  containers:
  - name: two-tier-app-cntr
    image: vaibhav9101/flaskapp:latest
    env:
      - name: MYSQL_HOST
        value: "mysql"
      - name: MYSQL_USER
        value: "root"
      - name: MYSQL_PASSWORD
        value: "admin"
      - name: MYSQL_DB
        value: "mydb"
    ports:
      - containerPort: 5000
    imagePullPolicy: Always


-----------------------------------------------------------------------------------------------------------------------------------------------------
kubectl apply -f pod.yml
------------------------------------------------------------------------------------------------------------------------------------------------------
vi deployment.yaml
----------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: two-tier-app-deployment
  labels:
    app: two-tier-app

spec:
  replicas: 3
  selector:
    matchLabels:
      app: two-tier-app
  template:
    metadata:
      labels:
        app: two-tier-app
    spec:
      containers:
        - name: two-tier-app-cont
          image: vaibhav9101/flaskapp:latest
          env:
            - name: MYSQL_HOST
              value: "mysql"
            - name: MYSQL_USER
              value: "root"
            - name: MYSQL_PASSWORD
              value: "admin"
            - name: MYSQL_DB
              value: "mydb"
          ports:
            - containerPort: 5000
          imagePullPolicy: Always 
------------------------------------------------------------------------------------------------------------------------------------------------------------
> kubectl apply -f deployment.yml
------------------------------------------------------------------------------------------------------------------------------------------------------------
> kubectl get pods
---------------------
To kill pods (@ Worker side)
_______________
> sudo docker kill <Container ID>
-----------------------------------------------------------------------------------------------------------------------------------------------------------
To Scale Up/Down Pods
______________________
>kubectl scale deployemnt two-tier-app-deployment --replicas=5

-----------------------------------------------------------------------------------------------------------------------------------------------------------

Services: 1. NodePort, 2. Cluster IP, 3. Load Balancer
-----------------------------------------------------------------------------------------------------------------------------------------------------------
vi service.yml
------------------
apiVersion: v1
kind: Service
metadata:
  name: two-tier-app-service
spec:
  selector:
    app: two-tier-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
      nodePort: 30008
  type: NodePort
----------------------------------------------------------------------------------------------------------------------------------------------------------
> kubectl apply -f service.yml
----------------------------------------------------------------------------------------------------------------------------------------------------------
kubectl get services
kubectl delete services two-tier-app-service (To delete service)
----------------------------------------------------------------------------------------------------------------------------------------------------------

For deploying MySQL database pod of Deployment kind
_____________________________________________________
vi DB_deployment.yml
---------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql

spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql-contr
          image: mysql:latest
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "admin"
            - name: MYSQL_USER
              value: "admin"
            - name: MYSQL_PASSWORD
              value: "admin"
            - name: MYSQL_DATABASE
              value: "mydb"
          ports:
            - containerPort: 3306
          imagePullPolicy: Always
------------------------------------------------------------------------------------------------------------------------------------------------------------
Persistent Volume 
___________________
pv.yml
--------------------
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 256Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/ubuntu/two-tier-flask-app/mysqldata 
------------------------------------------------------------------------------------------------------------------------------------------------------------
kubectl apply -f pv.yml
------------------------------------------------------------------------------------------------------------------------------------------------------------
Persistent Volume Claim
------------------------------
pvc.yml
----------
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi
------------------------------------------------------------------------------------------------------------------------------------------------------------
kubectl apply -f pvc.yml
------------------------------------------------------------------------------------------------------------------------------------------------------------
db_deployment.yml
------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:latest
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "admin"
            - name: MYSQL_DATABASE
              value: "mydb"
            - name: MYSQL_USER
              value: "admin"
            - name: MYSQL_PASSWORD
              value: "admin"
            
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysqldata
              mountPath: /var/lib/mysql
      volumes:
        - name: mysqldata
          persistentVolumeClaim:
            claimName: mysql-pvc
------------------------------------------------------------------------------------------------------------------------------------------------------------
kubectl apply -f db_deployment.yml
------------------------------------------------------------------------------------------------------------------------------------------------------------
ubuntu@ip-172-31-35-124:~$ kubectl get po
NAME                                       READY   STATUS    RESTARTS   AGE
mysql-c5b44f89c-zxcck                      1/1     Running   0          17m
two-tier-app-deployment-79dfb66cb7-5z49f   1/1     Running   0          19s
two-tier-app-deployment-79dfb66cb7-l6kmn   1/1     Running   0          16s
two-tier-app-deployment-79dfb66cb7-vnnm5   1/1     Running   0          24s
two-tier-app-pod                           1/1     Running   0          83m
------------------------------------------------------------------------------------------------------------------------------------------------------------
vi mysql_svc.yml
------------------------------------------------------------------------------------------------------------------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
------------------------------------------------------------------------------------------------------------------------------------------------------------
kubectl apply -f mysqlsvc.yml
------------------------------------------------------------------------------------------------------------------------------------------------------------
kubectl get svc
ubuntu@ip-172-31-35-124:~$ kubectl get svc
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes             ClusterIP   10.96.0.1       <none>        443/TCP        96m
mysql                  ClusterIP   10.96.157.113   <none>        3306/TCP       9s
                                  --------------
two-tier-app-service   NodePort    10.99.119.60    <none>        80:30008/TCP   39m
------------------------------------------------------------------------------------------------------------------------------------------------------------
vi deployment.yaml
------------------------------------------------------------------------------------------------------------------------------------------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: two-tier-app-deployment
  labels:
    app: two-tier-app

spec:
  replicas: 3
  selector:
    matchLabels:
      app: two-tier-app
  template:
    metadata:
      labels:
        app: two-tier-app
    spec:
      containers:
        - name: two-tier-app-cont
          image: vaibhav9101/flaskapp:latest
          env:
            - name: MYSQL_HOST
              value: "10.96.157.113"
            - name: MYSQL_USER
              value: "root"
            - name: MYSQL_PASSWORD
              value: "admin"
            - name: MYSQL_DB
              value: "mydb"
          ports:
            - containerPort: 5000
          imagePullPolicy: Always 
------------------------------------------------------------------------------------------------------------------------------------------------------------
kubectl apply -f deployment.yml
------------------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------------------------------------------
Go to Worker Nodes
------------------------------------------------------------------------------------------------------------------------------------------------------------
sudo docker ps
ubuntu@ip-172-31-33-99:~$ sudo docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED             STATUS             PORTS     NAMES
c49c1e86f53d   vaibhav9101/flaskapp    "python app.py"          2 minutes ago       Up 2 minutes                 k8s_two-tier-app-cont_two-tier-app-deployment-79dfb66cb7-l6kmn_default_5a772cb1-46aa-4d20-9279-6114d384eabb_0
16fd8e028ada   k8s.gcr.io/pause:3.2    "/pause"                 2 minutes ago       Up 2 minutes                 k8s_POD_two-tier-app-deployment-79dfb66cb7-l6kmn_default_5a772cb1-46aa-4d20-9279-6114d384eabb_0
4a3c50bc962e   vaibhav9101/flaskapp    "python app.py"          2 minutes ago       Up 2 minutes                 k8s_two-tier-app-cont_two-tier-app-deployment-79dfb66cb7-5z49f_default_e4c1c4e0-1e20-48bf-b664-06f386bc8f0e_0
dd0574fb21a2   k8s.gcr.io/pause:3.2    "/pause"                 2 minutes ago       Up 2 minutes                 k8s_POD_two-tier-app-deployment-79dfb66cb7-5z49f_default_e4c1c4e0-1e20-48bf-b664-06f386bc8f0e_0
57227ba923c0   vaibhav9101/flaskapp    "python app.py"          2 minutes ago       Up 2 minutes                 k8s_two-tier-app-cont_two-tier-app-deployment-79dfb66cb7-vnnm5_default_20661a56-41f4-4841-bb06-b2ac63f0bd09_0
58ca31d6d9ba   k8s.gcr.io/pause:3.2    "/pause"                 2 minutes ago       Up 2 minutes                 k8s_POD_two-tier-app-deployment-79dfb66cb7-vnnm5_default_20661a56-41f4-4841-bb06-b2ac63f0bd09_0
40d245367035   mysql                   "docker-entrypoint.s…"   19 minutes ago      Up 19 minutes                k8s_mysql_mysql-c5b44f89c-zxcck_default_b7f7cedc-03e0-4ded-ae9f-5c8ced44e929_0
20be40b2766a   k8s.gcr.io/pause:3.2    "/pause"                 19 minutes ago      Up 19 minutes                k8s_POD_mysql-c5b44f89c-zxcck_default_b7f7cedc-03e0-4ded-ae9f-5c8ced44e929_0
290b0a0a041b   vaibhav9101/flaskapp    "python app.py"          About an hour ago   Up About an hour             k8s_two-tier-app-cntr_two-tier-app-pod_default_f656e0b3-f9dd-47ce-beb8-838ac0ea5fe9_0
2dbb9b52d781   k8s.gcr.io/pause:3.2    "/pause"                 About an hour ago   Up About an hour             k8s_POD_two-tier-app-pod_default_f656e0b3-f9dd-47ce-beb8-838ac0ea5fe9_0
b5fd7f0fd9ca   weaveworks/weave-npc    "/usr/bin/launch.sh"     About an hour ago   Up About an hour             k8s_weave-npc_weave-net-ql7tt_kube-system_5a613143-095a-4003-8663-282d3678dbce_0
8f644d382607   weaveworks/weave-kube   "/home/weave/launch.…"   About an hour ago   Up About an hour             k8s_weave_weave-net-ql7tt_kube-system_5a613143-095a-4003-8663-282d3678dbce_0
2394e8884e93   k8s.gcr.io/kube-proxy   "/usr/local/bin/kube…"   About an hour ago   Up About an hour             k8s_kube-proxy_kube-proxy-f6qg9_kube-system_53b58ba8-761f-4ec6-a050-9e6d5c84b444_0
251b917cb524   k8s.gcr.io/pause:3.2    "/pause"                 About an hour ago   Up About an hour             k8s_POD_weave-net-ql7tt_kube-system_5a613143-095a-4003-8663-282d3678dbce_0
461e622179d0   k8s.gcr.io/pause:3.2    "/pause"                 About an hour ago   Up About an hour             k8s_POD_kube-proxy-f6qg9_kube-system_53b58ba8-761f-4ec6-a050-9e6d5c84b444_0
------------------------------------------------------------------------------------------------------------------------------------------------------------
Search for mysql container and enter in it.
--------------------------------------------
sudo docker exec -it 40d245367035 bash

bash-4.4# mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.2.0 MySQL Community Server - GPL

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mydb               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)

mysql> use mydb
Database changed
mysql> CREATE TABLE messages (
    ->     id INT AUTO_INCREMENT PRIMARY KEY,
    ->     message TEXT
    -> );
Query OK, 0 rows affected (0.02 sec)

mysql> select * from messages;
------------------------------------------------------------------------------------------------------------------------------------------------------------
Go to Web browser -> Enter IP of Worker Node followed by NodePort <http://13.233.173.239:30008/>
------------------------------------------------------------------------------------------------------------------------------------------------------------
Now you can see website and enter messages in it. 
------------------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------------------------------------------
HELM Packaging of Two-Tier Applications
------------------------------------------------------------------------------------------------------------------------------------------------------------
Install Helm
------------------------------------------------------------------------------------------------------------------------------
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
------------------------------------------------------------------------------------------------------------------------------------------------------------
To package 2-tier-app
-----------------------
mkdir two-tier-app
cd two-tier-app
Goto values.yaml
Edit following values
---------------------
replicaCount: 1  (Line no. 5)

image:
  repository: mysql  (Line no. 8)
  tag: "latest"     (Line no. 11)

service:
  type: ClusterIP
  port: 3306       (Line no. 44)
--------------------------------------------------------------------------------------------
To make a templete
--------------------------------------------------
Go to github
Go to two-tier-flask-app/k8s/mysql-deployment.yml
Copy Environment variables
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "admin"
            - name: MYSQL_DATABASE
              value: "mydb"
            - name: MYSQL_USER
              value: "admin"
            - name: MYSQL_PASSWORD
              value: "admin"
-------------------------------------------------------
Go to templates/deployment.yaml 
Go to line no. 38
Under  imagePullPolicy: {{ .Values.image.pullPolicy }}
Paste Environment variables on (Line no. 39)
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "admin"
            - name: MYSQL_DATABASE
              value: "mydb"
            - name: MYSQL_USER
              value: "admin"
            - name: MYSQL_PASSWORD
              value: "admin"
-------------------------------------------------------
Go to values.yaml file
Under tag: latest (Line no.11), write following values
env:
  mysqlrootpw: admin
  mysqldb: mydb
  mysqluser: admin
  mysqlpass: admin
--------------------------------------------------------
vi templates/deployment.yaml 
Make the following changes

imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: {{ .Values.env.mysqlrootpw }}
            - name: MYSQL_DATABASE
              value: {{ .Values.env.mysqldb }}
            - name: MYSQL_USER
              value: {{ .Values.env.mysqluser }}
            - name: MYSQL_PASSWORD
              value: {{ .Values.env.mysqlpass }}

& Save the file
---------------------------------------------------------
ubuntu@ip-172-31-11-121:~/two-tier-app$ helm package mysql-chart/
ubuntu@ip-172-31-11-121:~/two-tier-app$ helm install mysql-chart ./mysql-chart

O/P=>
NAME: mysql-chart
LAST DEPLOYED: Fri Nov 10 15:08:05 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=mysql-chart,app.kubernetes.io/instance=mysql-chart" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:$CONTAINER_PORT
---------------------------------------------------------
helm uninstall mysql-chart       => To uninstall HELM Package
-------------------------------------------------------------------------------------------------------------------------------------------
kubectl get all

o/P=>
NAME                               READY   STATUS    RESTARTS   AGE
pod/mysql-chart-86795bf6f9-57xpd   0/1     Running   0          18s

NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    62m
service/mysql-chart   ClusterIP   10.104.249.89   <none>        3306/TCP   18s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql-chart   0/1     1            0           18s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/mysql-chart-86795bf6f9   1         1         0       18s
-------------------------------------------------------------------------------------------------------------------------------------------
As we can see, the POD is running but not in READY state. We need to make some lines comment in deployment.yaml file
-------------------------------------------------------------------------------------------------------------------------------------------
 # livenessProbe:
                # httpGet:
                # path: /
                # port: http
                # readinessProbe:
                # httpGet:
                # path: /
                # port: http
-------------------------------------------------------------------------------------------------------------------------------------------
Again Uninstall, re-package and install package
---------------------------------------------------------
ubuntu@ip-172-31-11-121:~/two-tier-app$ kubectl get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/mysql-chart-cf7776b67-zdzqm   1/1     Running   0          11s

NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    81m
service/mysql-chart   ClusterIP   10.103.111.104   <none>        3306/TCP   11s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql-chart   1/1     1            1           11s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/mysql-chart-cf7776b67   1         1         1       11s
-------------------------------------------------------------------------------------------------------------------------------------------
-------------------------------------------------------------------------------------------------------------------------------------------
-------------------------------------------------------------------------------------------------------------------------------------------
Deploy flaskapp deployment
-------------------------------------------------------------------------------------------------------------------------------------------
ubuntu@ip-172-31-11-121:~$ cd two-tier-app/
ubuntu@ip-172-31-11-121:~/two-tier-app$ helm create flask-app-chart
ubuntu@ip-172-31-11-121:~/two-tier-app/flask-app-chart$ kubectl get svc
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    122m
mysql-chart   ClusterIP   10.103.111.104   <none>        3306/TCP   40m
-------------------------------------------------------------------------------------------------------------------------------------------
Go to values.yaml
Edit following values

replicaCount: 1

image:
  repository: vaibhav9101/flaskapp
 
  tag: "latest"

env:
  mysqlhost: 10.103.111.104 
  mysqlpw: admin
  mysqluser: admin
  mysqldb: mydb

service:
  type: NodePort
  port: 80
  targetPort: 5000
  nodePort: 30007
-----------------------------------------------------------------------------------------------------------------------------------------------
Go to deployment.yaml file
Edit following entries

         imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            - name: MYSQL_HOST
              value: {{ .Values.env.mysqlhost }}
            - name: MYSQL_PASSWORD
              value: {{ .Values.env.mysqlpw }}
            - name: MYSQL_USER
              value: {{ .Values.env.mysqluser }}
            - name: MYSQL_DB
              value: {{ .Values.env.mysqldb }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
-----------------------------------------------------------------------------------------------------------------------------------------------
Comment down following lines
                # livenessProbe:
                #httpGet:
                #path: /
                #port: http
                #readinessProbe:
                #httpGet:
                #path: /
                #port: http
-----------------------------------------------------------------------------------------------------------------------------------------------
Go to templates/service.yaml 
Edit following lines

spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      nodePort: {{ .Values.service.nodePort }}
      protocol: TCP
      name: http
-----------------------------------------------------------------------------------------------------------------------------------------------
Make a templete out of configuration

ubuntu@ip-172-31-11-121:~/two-tier-app$ helm templete flask-app-chart/
-----------------------------------------------------------------------------------------------------------------------------------------------
Make a package out of configuration

ubuntu@ip-172-31-11-121:~/two-tier-app$ helm package flask-app-chart/
-----------------------------------------------------------------------------------------------------------------------------------------------
ubuntu@ip-172-31-11-121:~/two-tier-app$ kubectl get all
NAME                                   READY   STATUS    RESTARTS   AGE
pod/flash-app-chart-64f9f6bf78-64w52   1/1     Running   0          46s
pod/flash-app-chart-64f9f6bf78-j4xqz   1/1     Running   0          46s
pod/flash-app-chart-64f9f6bf78-vv6w8   1/1     Running   0          45s
pod/mysql-chart-cf7776b67-zdzqm        1/1     Running   0          87m

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/flash-app-chart   NodePort    10.101.196.186   <none>        80:30007/TCP   46s
service/kubernetes        ClusterIP   10.96.0.1        <none>        443/TCP        169m
service/mysql-chart       ClusterIP   10.103.111.104   <none>        3306/TCP       87m

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/flash-app-chart   3/3     3            3           46s
deployment.apps/mysql-chart       1/1     1            1           87m
-----------------------------------------------------------------------------------------------------------------------------------------------
Now try to access webpage by using Worker Node IP

http://15.206.174.13:30007/
-----------------------------------------------------------------------------------------------------------------------------------------------
Go to Worker Node

sudo docker ps
ubuntu@ip-172-31-1-24:~$ sudo docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED          STATUS          PORTS     NAMES
3da071bbc0ff   a3b6608898d6            "docker-entrypoint.s…"   35 minutes ago   Up 35 minutes             k8s_mysql-chart_mysql-chart-cf7776b67-zdzqm_default_23a2f006-eadf-47ea-bcec-a06c7be23013_0
cbb31c62cc1d   k8s.gcr.io/pause:3.2    "/pause"                 35 minutes ago   Up 35 minutes             k8s_POD_mysql-chart-cf7776b67-zdzqm_default_23a2f006-eadf-47ea-bcec-a06c7be23013_0
e99a4a98c8da   weaveworks/weave-npc    "/usr/bin/launch.sh"     2 hours ago      Up 2 hours                k8s_weave-npc_weave-net-krx2x_kube-system_00239a71-ade8-4b62-b8f4-ac733b7faf8b_0
b1f92045f7e9   weaveworks/weave-kube   "/home/weave/launch.…"   2 hours ago      Up 2 hours                k8s_weave_weave-net-krx2x_kube-system_00239a71-ade8-4b62-b8f4-ac733b7faf8b_0
4156c4945799   k8s.gcr.io/kube-proxy   "/usr/local/bin/kube…"   2 hours ago      Up 2 hours                k8s_kube-proxy_kube-proxy-dxns7_kube-system_413bd06d-be3b-403a-9716-44e4345886b3_0
4b69234dad97   k8s.gcr.io/pause:3.2    "/pause"                 2 hours ago      Up 2 hours                k8s_POD_weave-net-krx2x_kube-system_00239a71-ade8-4b62-b8f4-ac733b7faf8b_0
150b59399c36   k8s.gcr.io/pause:3.2    "/pause"                 2 hours ago      Up 2 hours                k8s_POD_kube-proxy-dxns7_kube-system_413bd06d-be3b-403a-9716-44e4345886b3_0
-----------------------------------------------------------------------------------------------------------------------------------------------
ubuntu@ip-172-31-1-24:~$ sudo docker exec -it 3da071bbc0ff bash
bash-4.4# mysql -u admin -p
Enter password: 
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mydb               |
| performance_schema |
+--------------------+

mysql> use mydb;
Database changed
mysql> CREATE TABLE messages (
    ->     id INT AUTO_INCREMENT PRIMARY KEY,
    ->     message TEXT
    -> );

-----------------------------------------------------------------------------------------------------------------------------------------------
Again try to access webpage

Put some message in webpage

Go to worker node

mysql> select * from messages;
+----+-------------------------------------------------+
| id | message                                         |
+----+-------------------------------------------------+
|  1 | Hiii, Hello, How are you, I am fine. Thank you! |
+----+-------------------------------------------------+
1 row in set (0.00 sec)
-----------------------------------------------------------------------------------------------------------------------------------------------
To delete webapp
----------------------
ubuntu@ip-172-31-11-121:~/two-tier-app$ helm list
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
flash-app-chart default         1               2023-11-10 17:04:17.693845814 +0000 UTC deployed        flash-app-chart-0.1.0   1.16.0     
mysql-chart     default         1               2023-11-10 15:37:10.308423205 +0000 UTC deployed        mysql-chart-0.1.0       1.16.0     
ubuntu@ip-172-31-11-121:~/two-tier-app$ helm uninstall flash-app-chart
release "flash-app-chart" uninstalled
-----------------------------------------------------------------------------------------------------------------------------------------------
-----------------------------------------------------------------------------------------------------------------------------------------------
EKS Cluster Setup & Deployment of Two-Tier Application 
-----------------------------------------------------------------------------------------------------------------------------------------------
1. Launch an Ubuntu Machine
2. Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update

3. Make an IAM User who has access of  
a. Creating Cluster
b. EC2 instance
c. Creating Roles
d. Bucket

4. Create IAM user and asign required permissions and generate ACCESS key
AKIAZ4ASDFLHRNTF7BWQ
j/v8Mzlk6cK0jmPdzjb9KeI9Cgqe//WYtXenXHNz

-----------------------------------------------------------------------------------------------------------------------------------------------
To install kubectl
---------------------
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.3/2023-11-02/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version 

To install eksctl
---------------------
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
-----------------------------------------------------------------------------------------------------------------------------------------------

Create Cluster
---------------------
eksctl create cluster --name tws-cluster --region ap-south-1 --node-type t3.small --nodes-min 2 --nodes-max 3

COPY deployment.yml, db_deployment.yml, service.yml, mysql_service.yml in Master Node
-----------------------------------------------------------------------------------------------------------------------------------------------

Create secret.yml
---------------------
apiVersion: v1
kind: Secret
metadata: 
  name: mysql-secret
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: "YWRtaW4="(For Password, follow the following steps)
--------------------------------------------------------------------------
Go to EC2 Instance & type

echo -n "admin" | base64
        ------
       Password

O/P=> YWRtaW4=
--------------------------------------------------------------------------

If I want that the database we create, it should automatically created, For that we will create ConfigMap.yaml file
                                                                                               ---------------      
-----------------------------------------------------------------------------------------------------------------------------------------------
configmap.yml
-------------------
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-initdb-config
data: 
  init.sql: |
    CREATE DATABASE IF NOT EXISTS mydb;
    USE mydb;  
    CREATE TABLE messages (id INT AUTO_INCREMENT PRIMARY KEY,message TEXT);
-----------------------------------------------------------------------------------------------------------------------------------------------

db_deployment.yml
------------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:latest
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "admin"
            - name: MYSQL_DATABASE
              value: "mydb"
            - name: MYSQL_USER
              value: "admin"
            - name: MYSQL_PASSWORD
              value: "admin"
            
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-initdb
              mountPath: docker-entrypoint-initdb.d
      volumes:   #Volumes injects/connect something in our PODs
        - name: mysql-initdb
          configMap:
            name: mysql-initdb-config  #ConfigName
-----------------------------------------------------------------------------------------------------------------------------------------------
ENTRYPOINT is one of the many instructions you can write in a dockerfile. The ENTRYPOINT instruction is used to configure the executables that will always run after the container is initiated. For example, you can mention a script to run as soon as the container is started.
The sql script available in the container path /docker-entrypoint-initdb.d will only be executed when /var/lib/mysql does not contain database data. Meaning: it is only intended for the initial setup.
-----------------------------------------------------------------------------------------------------------------------------------------------
vi deployment.yml
---------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: two-tier-app-deployment
  labels:
    app: two-tier-app

spec:
  replicas: 3
  selector:
    matchLabels:
      app: two-tier-app
  template:
    metadata:
      labels:
        app: two-tier-app
    spec:
      containers:
        - name: two-tier-app-cont
          image: vaibhav9101/flaskapp:latest
          env:
            - name: MYSQL_HOST
              value: mysql               # Change this name same as mysql_service file's metadata name
            - name: MYSQL_USER
              value: "root"
            - name: MYSQL_PASSWORD
              value: "admin"
            - name: MYSQL_DB
              value: "mydb"
          ports:
            - containerPort: 5000
          imagePullPolicy: Always 
-----------------------------------------------------------------------------------------------------------------------------------------------
vi two-tier-app-service.yml
----------------------------
apiVersion: v1
kind: Service
metadata:
  name: two-tier-app-service
spec:
  selector:
    app: two-tier-app
  type: LoadBalancer         #Removed Nodeport and add Load balancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
-----------------------------------------------------------------------------------------------------------------------------------------------
To delete cluster
eksctl delete cluster --name tws-cluster-1 --region ap-south-1