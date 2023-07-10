# kubernetes-twoge
My task on this project was to deploy the twoge application utilizing kubernete's locally via minikube and then on AWS Elastic Kubernetes Service . 
<img src="https://github.com/prestonmoody/kubernetes-twoge/blob/main/Local-to-EKS.png">

First I git cloned chandras repository of the twoge application which can be found @ https://github.com/chandradeoarya/twoge/tree/k8s

I then created a Dockerfile and built an image.
```
FROM python:alpine


RUN apk update && \
    apk add --no-cache build-base libffi-dev openssl-dev
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 8080
CMD python app.py
```
I then pushed this image to my Dockerhub which I will reference in my deployment yamel file

```
docker push pmoody199/twoge-k8s
```
I then wrote 6 .yml files for the app, app service, database, database service, secret, and config map.

In this deployment file you will notice my readiness probe that has an initial delay to wait for my postgres database pod to form before creating itself.

twoge-dep.yml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: twoge-dep
  labels:
    app: twoge-k8s
spec:
  selector:
    matchLabels:
      app: twoge-k8s
  replicas: 1
  template:
    metadata:
      labels:
        app: twoge-k8s
    spec:
      containers:
      - name: twoge-container
        image: pmoody199/twoge-k8s
        ports:
          - containerPort: 8080
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        env:
          - name: DB_DATABASE
            valueFrom:
              secretKeyRef:
                name: postgres-secret
                key: postgres-database
          - name: DB_USER
            valueFrom:
              secretKeyRef:
                name: postgres-secret
                key: postgres-username
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: postgres-secret
                key: postgres-password
          - name: DB_HOST
            valueFrom:
              configMapKeyRef:
                name: postgres-configmap
                key: database_host
          - name: DB_PORT
            valueFrom:
              configMapKeyRef:
                name: postgres-configmap
                key: database_port

```

twoge-service.yml
```
apiVersion: v1
kind: Service
metadata:
  name: twoge-k8s-service
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30082
  selector:
    app: twoge-k8s
```
postgres-dep.yml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres
        # volumeMounts:
        #   - mountPath: /var/lib/postgres
        #     name: postgres-storage
        env:
          - name: POSTGRES_DB
            valueFrom:
              secretKeyRef:
                name: postgres-secret
                key: postgres-database
          - name: POSTGRES_USER
            valueFrom:
              secretKeyRef:
                name: postgres-secret
                key: postgres-username
          - name: POSTGRES_PASSWORD
            valueFrom:
              secretKeyRef:
                name: postgres-secret
                key: postgres-password
      # volumes:
      #   - name: postgres-storage
      #     hostPath:
      #       path: /data-twoge
      #       type: Directory
```
postgres-service.yml
```
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  type: ClusterIP
  ports:
  - port: 5432
```
postgres-secrets.yml
```
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  postgres-database: ****
  postgres-username: ****
  postgres-password: ****

```
postgres-configmap.yml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-configmap
data:
  database_host: "postgres-service"
  database_port: "5432"
```
I then created a new name space for seperate testing called twogespace via a .yml file

namespace-twoge.yml
```
apiVersion: v1
kind: Namespace
metadata:
  name: twogespace
  labels:
    name: twogespace
```
I then redeployed all of my files in this name space
```
kubectl apply -f . --namespace twogespace
```
Next I created a resourcequota
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-quota
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```
I then went into my EC2 instance and brought all of my yml files with me.

I made a new directory separate from where my config file for the EKS stack lived to keeps commands simple.

Next I changed my namespace for the EKS stack to my own
```
kubectl config set-context --current --namespace=preston
```
I then ran all of the same commands while only changing my app service file type to LoadBalancer and deleting the NodePort since we will no longer need it.
```
apiVersion: v1
kind: Service
metadata:
  name: twoge-k8s-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: twoge-k8s
```
challenges.
1. Switching the nodeports between namespaces.
2. The order of deploying these .yml files especially the resource quota.
3. Getting the EKS stack configured.










