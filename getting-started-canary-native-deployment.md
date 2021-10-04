# Canary deployment using Kubernetes native functionnalities

# Steps to follow
1. 4 replicas of version 1 is serving traffic
2. deploy 1 replicas version 2 (meaning ~25% of traffic)
3. wait enought time to confirm that version 2 is stable and not throwing unexpected errors
4. scale up version 2 replicas to 4
5. wait until all instances are ready
6. shutdown version

# DEMO, Lets say we have deployment file as app-v1.yaml and app-v2.yaml 
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ cat app-v1.yaml 
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: http
  selector:
    app: my-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v1
  labels:
    app: my-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: my-app
      version: v1.0.0
  template:
    metadata:
      labels:
        app: my-app
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9101"
    spec:
      containers:
      - name: my-app
        image: containersol/k8s-deployment-strategies
        ports:
        - name: http
          containerPort: 8080
        - name: probe
          containerPort: 8086
        env:
        - name: VERSION
          value: v1.0.0
        livenessProbe:
          httpGet:
            path: /live
            port: probe
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: probe
          periodSeconds: 5
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ 

# save and quit

(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ cat app-v2.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v2
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      version: v2.0.0
  template:
    metadata:
      labels:
        app: my-app
        version: v2.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9101"
    spec:
      containers:
      - name: my-app
        image: containersol/k8s-deployment-strategies
        ports:
        - name: http
          containerPort: 8080
        - name: probe
          containerPort: 8086
        env:
        - name: VERSION
          value: v2.0.0
        livenessProbe:
          httpGet:
            path: /live
            port: probe
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: probe
          periodSeconds: 5
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ 

# save and quit

# Deploy the first application
$ kubectl apply -f app-v1.yaml

# Test if the deployment was successful
$ curl $(minikube service my-app --url)
# OR
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ minikube ssh
Last login: Thu Sep 30 04:29:07 2021 from 192.168.49.1
docker@minikube:~$ curl http://192.168.49.2:30623
Host: my-app-v1-847db5bd98-d2w8v, Version: v1.0.0
docker@minikube:~$ 

# To see the deployment in action, open a new terminal and run a watch command.
# It will show you a better view on the progress
$ watch kubectl get po

(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
my-app-v1-847db5bd98-d2w8v   1/1     Running   0          7m22s
my-app-v1-847db5bd98-pnfp2   1/1     Running   0          7m22s
my-app-v2-6547f7c7b4-wcwzm   1/1     Running   0          7m9s

# Then deploy version 2 of the application and scale down version 1 to 2 replicas at same time
$ kubectl apply -f app-v2.yaml
$ kubectl scale --replicas=2 deploy my-app-v1

# Testing ..

(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ kubectl scale --replicas=2 deploy my-app-v2
deployment.apps/my-app-v2 scaled
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
my-app-v1-847db5bd98-d2w8v   1/1     Running   0          7m44s
my-app-v1-847db5bd98-pnfp2   1/1     Running   0          7m44s
my-app-v2-6547f7c7b4-92bgm   1/1     Running   0          4s
my-app-v2-6547f7c7b4-wcwzm   1/1     Running   0          7m31s
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ minikube ssh
Last login: Thu Sep 30 22:39:12 2021 from 192.168.49.1
docker@minikube:~$ 
docker@minikube:~$ curl http://192.168.49.2:30623
Host: my-app-v1-847db5bd98-d2w8v, Version: v1.0.0
docker@minikube:~$ curl http://192.168.49.2:30623
Host: my-app-v1-847db5bd98-d2w8v, Version: v1.0.0

# More Testing ..

(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ kubectl scale --replicas=2 deploy my-app-v2
deployment.apps/my-app-v2 scaled
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ 
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/canary-dep$ minikube ssh
Last login: Thu Sep 30 23:04:57 2021 from 192.168.49.1
docker@minikube:~$ while sleep 0.1; do curl http://192.168.49.2:30623; done
Host: my-app-v2-6547f7c7b4-wcwzm, Version: v2.0.0
Host: my-app-v1-847db5bd98-9g8dz, Version: v1.0.0
Host: my-app-v2-6547f7c7b4-92bgm, Version: v2.0.0

