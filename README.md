# Deploy-a-Webserver-DevOps
This is a Junior level DevOps project which deploys local environment NGINX webserver and displays a Hello world message on a Kubernetes cluster. Additionally, Docker is used for containerization and Helm acts as a local chart.

## Preparation & Tools installation
Install docker desktop, install minikube, install helm;
```
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-darwin-amd64
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
```

## Infrastructure setup
Deploy a local Kubernetes cluster with ingress capabilities, to able to access it locally via the local ingress domain (NGINX).
```
minikube start
minikube addons enable ingress
```
To verify use `kubectl get pods -n ingress-nginx`
```
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-kq5xv        0/1     Completed   0          9m29s
ingress-nginx-admission-patch-2jz8v         0/1     Completed   1          9m29s
ingress-nginx-controller-56d7c84fd4-67vv9   1/1     Running     0          9m29s
```

## Webserver and Docker
Create a Dockerfile that pulls an nginx image and configure it so it launches as a foreground process. The webserver should  print “Hello world” when visited. Additionally use Kubernetes ConfigMap and make sure Kubernetes cluster can pull the image

Create a dockerfile:
```
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
Create a index.html for web:
```
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Webserver by Edvinas using Docker Nginx</title>
</head>
<body>
  <h2>Hello world</h2>
</body>
</html>
```
```
cd edvinas-k8-webserver 
docker build -t edvinas-nginx-hello-world .
docker run -d -p 8080:80 edvinas-nginx-hello-world
```
To verify use `docker ps` or go to your localhost in browser
```
CONTAINER ID   IMAGE                                 COMMAND                  CREATED             STATUS             PORTS                                                                                                                                  NAMES
4466539be886   gcr.io/k8s-minikube/kicbase:v0.0.46   "/usr/local/bin/entr…"   About an hour ago   Up About an hour   127.0.0.1:50859->22/tcp, 127.0.0.1:50860->2376/tcp, 127.0.0.1:50857->5000/tcp, 127.0.0.1:50858->8443/tcp, 127.0.0.1:50856->32443/tcp   minikube
```
Kubernetes ConfigMap (bonus):

Create a config-file.yaml:
```
apiVersion: v1
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <title>Testing ConfigMap - Edvinas</title>
    </head>
    <body>
        <p>Hello world<p>
    </body>
    </html>
kind: ConfigMap
metadata:
  name: my-config
  namespace: default
```
Create a deployment.yaml:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
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
          image: edvinas-nginx-hello-world
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /usr/share/nginx/html
      volumes:
        - name: nginx-config
          configMap:
            name: my-config
```
To deploy the files, setup loadbalancer and verify connection, use:
```
kubectl apply -f config-file.yaml
kubectl get configmap
NAME               DATA   AGE
kube-root-ca.crt   1      3h8m
my-config          1      7s
```
```
kubectl create -f deployment.yaml 
kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           11s
```
```
kubectl expose deployment/nginx-deployment --port 80 --name nginx-service-lb --type LoadBalancer
service/nginx-service-lb exposed
kubectl get svc                                  
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP      10.96.0.1        <none>        443/TCP        3h16m
nginx-service-lb   LoadBalancer   10.105.218.177   <pending>     80:30168/TCP   4m12s
```
```
kubectl port-forward svc/nginx-service-lb 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```
To upload image to Kubernetes cluster and to verify use:
```
minikube image load edvinas-nginx-hello-world
kubectl apply -f deployment.yaml
minikube dashboard
```
## Helm and Deployment
Template local Helm chart which would include all required resources to deploy the webserver and make it accessible, make it also accessible via local ingress domain
Create a chart using `helm create nginx-chart`
Fix yaml files and remove junk:
Edit Chart.yaml:
```
apiVersion: v2
name: nginx-chart
description: A Helm chart for Kubernetes - Nord Security
type: application
version: 0.1.0
appVersion: "1.27.4"
maintainers:
- email: edvinas.vencevicius@gmail.com
  name: edvinasvencevicius
```
Edit values.yaml:
```
replicaCount: 1

image:
  repository: edvinas-nginx-hello-world
  tag: latest
  pullPolicy: IfNotPresent

service:
  name: nginx-service
  type: ClusterIP
  port: 80
  targetPort: 80

env:
  name: dev

ingress:
  enabled: true
  hostname: localhost
  path: /
  annotations: {}
```
Edit remaining files: `configmap.yaml` `deployment.yaml` `ingress.yaml` `service.yaml` (skipping source codes here - available in repo)

To validate, deploy helm chart and verify use:
```
helm lint .               
==> Linting .
[INFO] Chart.yaml: icon is recommended
1 chart(s) linted, 0 chart(s) failed
```
```
helm list
NAME    	NAMESPACE	REVISION	UPDATED                              	STATUS  	CHART            	APP VERSION
frontend	default  	1       	2025-04-10 07:55:55.619814 +0300 EEST	deployed	nginx-chart-0.1.0	1.27.4
```   
```
kubectl get deployment      
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     2/2     2            2           10h
release-name-nginx   1/1     1            1           55s
```
```
kubectl get services
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
frontend-service   ClusterIP      10.105.128.10    <none>        80/TCP         96s
kubernetes         ClusterIP      10.96.0.1        <none>        443/TCP        13h
nginx-service-lb   LoadBalancer   10.105.218.177   <pending>     80:30168/TCP   10h
```
```
kubectl get configmap
NAME                            DATA   AGE
frontend-index-html-configmap   1      104s
kube-root-ca.crt                1      13h
my-config                       1      10h
```
```
kubectl get pods
NAME                                  READY   STATUS    RESTARTS     AGE
nginx-deployment-54f8f67c48-r6r8k     1/1     Running   1 (9h ago)   10h
nginx-deployment-54f8f67c48-vzw5j     1/1     Running   1 (9h ago)   10h
release-name-nginx-5d57599885-m4v7c   1/1     Running   0            107s
```
```
kubectl get ingress
NAME            CLASS   HOSTS       ADDRESS        PORTS   AGE
nginx-ingress   nginx   localhost   192.168.49.2   80      3m23s
```
To verify use either:

port forwarding - `kubectl port-forward svc/nginx-service-lb 8080:80` and access your localhost

Ingress - make sure /etc/hosts has added line `127.0.0.1    nginx.local` (or localhost)

## Final note and checklist

<ins>Kubernetes cluster must be accessible</ins>

<ins>Dockerfile/nginx image in local host it should print Hello world</ins>

<ins>Docker image must be loaded into minikube</ins>

<ins>Helm chart is created with needed files and does not fail</ins>

<ins>We can access our web/app using localhost or nginx.local</ins>
