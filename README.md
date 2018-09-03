## Setting in GCP

1. Create cluster in GCP


2. Open Cloud Shell on GCP and run following commands for setting.

```yml
sudo gcloud components install kubectl

export PROJECT_ID="$(gcloud config get-value project -q)"

gcloud container clusters get-credentials cluster-3 --region asia-east1-a
```
 
## Overview of Example

* Step 1. create volume for mysql deployment.

* Step 2. create mysql deployment.

* Step 3. create mysql service.

* Step 4. create redis deployment.

* Step 5. create redis service.

* Step 6. create spring deployment.

* Step 7. create spring service.


### Step 1. Create Volume For Mysql Deployment.


There is a yaml file **my-mysql-pvc.yaml** that creates Persistent volumes.

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-volumeclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```
my-mysql-pvc.yaml would be used by my-mysql-deployment.yaml

```sh
kubectl create -f my-mysql-pvc.yaml
```

Check out the persistent volume claim.

```sh
kubectl get pvc

NAME                STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-volumeclaim   Bound     pvc-deac2352-af39-11e8-ad37-42010a8c00c9   3Gi        RWO            standard       3h
```


If you want to delete the persistent volume claim.

(you can execute following command after fininshing all example.)

```sh
kubectl delete pvc  mysql-volumeclaim
```


### Step 2. Create Mysql Deployment.

Based on pvc, you can create a mysql pod and mount data on volume.

```yaml
apiVersion: apps/v1beta2 # for kubectl versions >= 1.9.0 use apps/v1
kind: Deployment
metadata:
  name: my-mysql-deployment
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-mysql-app
  template:
    metadata:
      labels:
        app: my-mysql-app
    spec:
      containers:
        - image: mysql:5.6
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "123456789"
            - name: MYSQL_DATABASE
              value: "mydb"
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: mysql-volumeclaim
```


1. As you can see, this deploymeny yaml file refers `mysql-volumeclaim` and mounts it on `/var/lib/mysql` of container.

2. selector has `my-mysql-app` that it would be used by my-mysql-service.

	*  service will forward traffic to `my-mysql-app`


Create a deployment

```sh
kubectl create -f my-mysql-service.yaml
```

Check out the deployment

```sh
kubectl get deployment

NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
my-mysql-deployment   1         1         1            1           1m
```

### Step 3. Create Mysql Service.

This specification will create a new Service object named **“my-mysql-service”** which targets TCP port **3306** on any Deployment with the **"app=my-mysql-app"** label

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-mysql-service
spec:
  ports:
  - port: 3306
    protocol: TCP
  selector:
    app: my-mysql-app
  type: ClusterIP
```

* **“my-mysql-service”** will recieve traffic and forward it to deployment with **"app=my-mysql-app"**.


Create a service

```sh
kubectl create -f my-mysql-service.yaml.
```

Check out the service

```sh
kubectl get service

NAME               TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
kubernetes         ClusterIP      10.11.240.1     <none>         443/TCP        7h
my-mysql-service   ClusterIP      10.11.242.82    <none>         3306/TCP       1m
```

### Step 4. Create Redis deployment.

Create a redis deployment that it would be used to store session of spring web application.

```yaml
apiVersion: apps/v1beta2 # for kubectl versions >= 1.9.0 use apps/v1
kind: Deployment
metadata:
  name: my-redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-redis-app
  template:
    metadata:
      labels:
        app: my-redis-app
    spec:
      containers:
      - name: my-redis
        image: redis
        ports:
        - containerPort: 6379
```

1. Give it a label named **my-redis-app**. Service will forward traffic to a deployment with **my-redis-app**.

2. redis uses port 6379.


create redis deployment

```sh
kubectl create -f my-redis-deployment.yaml
```

Check out the deployment

```sh
kubectl get deployment

NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
my-mysql-deployment   1         1         1            1           2m
my-redis-deployment   1         1         1            1           1m
```


### Step 5. create redis service.


create redis service

```sh
kubectl create -f my-redis-service.yaml
```

Check out the service

```sh
kubectl get service

NAME               TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
kubernetes         ClusterIP      10.11.240.1     <none>         443/TCP        8h
my-mysql-service   ClusterIP      10.11.242.82    <none>         3306/TCP       4h
my-redis-service   ClusterIP      10.11.247.76    <none>         6379/TCP       3h
```


### Step 6. Create Spring Deployment.

1. Web Application is developed by Spring Boot.

2. Spring Boot Image is from My Docker Hub.

3. This web application uses MySQL and Redis

4. Redis is used to store session.

```yaml
apiVersion: apps/v1beta2 # for kubectl versions >= 1.9.0 use apps/v1
kind: Deployment
metadata:
  name: my-web-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-deployment
  template:
    metadata:
      labels:
        app: my-deployment
    spec:
      containers:
      - name: my-spring
        image: lucasko/spring-boot-stateless
        ports:
        - containerPort: 8080
        env:
        - name: REDIS_HOST
          value: "my-redis-service"
        - name: MYSQL_HOST
          value: "my-mysql-service"
        - name: MYSQL_DB
          value: "mydb"
        - name: MYSQL_USERNAME
          value: "root"
        - name: MYSQL_PASSWORD
          value: "123456789"
```

 * REDIS_HOST is `my-redis-service`. The redis service will forward connection to my-redis-deployment

 * MYSQL_DB is `my-mysql-service`. The mysql service will forward connection to my-mysql-deployment


```sh
kubectl create -f my-web-deployment.yaml
```


### Step 7. create spring service.

Create a service with LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-web-service
spec:
  type: LoadBalancer
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8080
  selector:
    app: my-deployment
status:
  loadBalancer:
    ingress:
    - ip: 60.248.176.103
```

 * Forward 80 port to pod's 8080 port.
 * type is LoadBalancer

create web service

```sh
kubectl create -f my-web-service.yaml
```

check out the service

```sh
kubectl get service

NAME               TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
kubernetes         ClusterIP      10.11.240.1     <none>         443/TCP        8h
my-mysql-service   ClusterIP      10.11.242.82    <none>         3306/TCP       4h
my-redis-service   ClusterIP      10.11.247.76    <none>         6379/TCP       4h
my-web-service     LoadBalancer   10.11.241.203   35.221.164.6   80:32251/TCP   4h
```


















