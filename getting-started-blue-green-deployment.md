# Blue/green deployment to release a single service
# Steps to follow
1. version 1 is serving traffic
2. deploy version 2
3. wait until version 2 is ready
4. switch incoming traffic from version 1 to version 2
5. shutdown version 1

# DEMO Let's suppose we have two version of file as app-v1.yaml and app-v2.yaml
vim app-v1.yaml

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

  #Note here that we match both the app and the version
  selector:
    app: my-app
    version: v1.0.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v1
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: v1.0.0
  template:
    metadata:
      labels:
        app: my-app
        version: v1.0.0
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

# Save and Quit

# Deploy the first application
$ kubectl apply -f app-v1.yaml

# Test if the deployment was successful
$ curl $(minikube service my-app --url)
# OR
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/blue-green-deployment$ minikube ssh
Last login: Thu Sep 30 03:59:23 2021 from 192.168.49.1
docker@minikube:~$ curl http://192.168.49.2:30623
Host: my-app-v1-6f677cb69c-wp6kh, Version: v1.0.0
docker@minikube:~$ exit

# To see the deployment in action, open a new terminal and run the following command:
$ watch kubectl get po

---------------------------
# Let's create another file as my-app-v2 with following yaml 
vim my-app-v2

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v2
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: v2.0.0
  template:
    metadata:
      labels:
        app: my-app
        version: v2.0.0
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
          
# Save and Quit

# Then deploy version 2 of the application
$ kubectl apply -f app-v2.yaml

# Wait for all the version 2 pods to be running
$ kubectl rollout status deploy my-app-v2 -w
deployment "my-app-v2" successfully rolled out

# Once you are ready, you can switch the traffic to the new version by patching
# the service to send traffic to all pods with label version=v2.0.0 as following ...
$ kubectl patch service my-app -p '{"spec":{"selector":{"version":"v2.0.0"}}}'
example: 
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/blue-green-deployment$ kubectl patch service my-app -p '{"spec":{"selector":{"version":"v2.0.0"}}}'
service/my-app patched
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/blue-green-deployment$

(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/blue-green-deployment$ minikube ssh
Last login: Thu Sep 30 04:22:07 2021 from 192.168.49.1
docker@minikube:~$ 
docker@minikube:~$ curl http://192.168.49.2:30623
Host: my-app-v2-5d6c7554b6-xtmg5, Version: v2.0.0
docker@minikube:~$ 

# In case you need to rollback to the previous version
$ kubectl patch service my-app -p '{"spec":{"selector":{"version":"v1.0.0"}}}'
Example: 

(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/blue-green-deployment$ kubectl patch service my-app -p '{"spec":{"selector":{"version":"v1.0.0"}}}'
service/my-app patched
(⎈ |minikube:default)bkapar@LM-SJC-11024580:~/minikube/blue-green-deployment$ minikube ssh
Last login: Thu Sep 30 04:27:57 2021 from 192.168.49.1
docker@minikube:~$ curl http://192.168.49.2:30623
Host: my-app-v1-6f677cb69c-wp6kh, Version: v1.0.0
docker@minikube:~$ 

# If everything is working as expected, you can then delete the v1.0.0
# deployment
$ kubectl delete deploy my-app-v1
# Cleanup
$ kubectl delete all -l app=my-app

# Happy Learning :) 


