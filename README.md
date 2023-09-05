# minikube-ansible-todo-app

Using Ansible playbook, set up a single Node Kubernetes cluster into a Virtual Machine to deploy the full stack of the complete "TODO" application provided by Docker in the following tutorial:
https://docs.docker.com/get-started/02_our_app/

Let's break down the configuration files in this repository starting with the deployment files:

Kubernetes Deployment for Todo Application:

This document outlines the Kubernetes resources used to deploy a Todo Application and its supporting infrastructure.

Overview
The application consists of two main components:

Redis - An in-memory data structure store, used as a database, cache, and message broker.
Todo-app - A web application to manage todos which communicates with Redis for data storage.
The Kubernetes resources utilized to deploy these components are:

PersistentVolume (PV) & PersistentVolumeClaim (PVC)
Deployments
Services
Ingress

PersistentVolume (PV) & PersistentVolumeClaim (PVC)
redis-pv
This is a PersistentVolume (PV) that provides 1Gi of storage on the host's filesystem at /mnt/data.

Usage: This PV is used to store data for the Redis database to ensure that data remains intact across container restarts.
```yaml

apiVersion: v1
kind: PersistentVolume
...
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  hostPath:
    path: "/mnt/data"

```

redis-pvc
This is a PersistentVolumeClaim (PVC) that requests storage from the redis-pv PersistentVolume.

Usage: The PVC is used by the Redis deployment to mount the storage provided by the PV.

```yaml

apiVersion: v1
kind: PersistentVolumeClaim
...
spec:
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi

```

Deployments:
Redis Deployment
Deploys a single replica of the redis:alpine image. This Redis instance uses the storage provisioned by the redis-pvc.

Key Configurations:
Volume Mount: The Redis container mounts the PVC at /data, which is typically where Redis stores its data.

```yaml
apiVersion: apps/v1
kind: Deployment
...
spec:
  containers:
    - name: redis
      image: redis:alpine
      volumeMounts:
        - name: redis-storage
          mountPath: /data
```

Todo-app Deployment
Deploys a single replica of the mmarius19/getting-started:latest image. This is the Todo web application.

Key Configurations:
Environment Variables: Configures the application to connect to the Redis service using the hostname redis on port 6379.

```yaml
apiVersion: apps/v1
kind: Deployment
...
spec:
  containers:
    - name: todo-app
      env:
        - name: REDIS_HOST
          value: redis
        - name: REDIS_PORT
          value: "6379"
```

Services
Redis Service
A ClusterIP service (default type) that provides internal connectivity to the Redis deployment within the Kubernetes cluster.

```yaml
apiVersion: v1
kind: Service
...
spec:
  ports:
    - port: 6379
```
Todo-app Service
A LoadBalancer service that provides external connectivity to the Todo web application. This means it will also provision a cloud load balancer when deployed on cloud providers that support this service type.

```yaml
apiVersion: v1
kind: Service
...
spec:
  type: LoadBalancer
  ports:
    - port: 3000
```
Ingress
todo-ingress
An Ingress resource that provides HTTP routing to the Todo web application. It allows external users to access the app using the hostname todo.local.

Key Configurations:

Path: The ingress routes traffic for the path / to the todo-app service on port 3000.
Rewrite Target Annotation: All requests are rewritten to / irrespective of the original path. This might be useful if the application expects all requests to hit the root path.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
...
spec:
  rules:
    - host: todo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: todo-app
                port:
                  number: 3000
```
Conclusion:

This setup ensures data persistence for Redis, scalable deployment for the web application, internal communication via services, and external access via a LoadBalancer and Ingress. 
Remember to map the domain (todo.local) to the appropriate IP address in your environment (e.g., /etc/hosts or DNS) for external access.

Now let's break down the Ansible Playbook

This Ansible playbook sets up Minikube, Docker, and deploys a Todo Application on a VM. The detailed breakdown of each task is as follows:


1. Ensure software-properties-common is installed:
This package is required to add external repositories using the apt-add-repository command.

```yml
- name: Ensure software-properties-common is installed (for apt-add-repository)
  apt:
    name: software-properties-common
    state: present
```

