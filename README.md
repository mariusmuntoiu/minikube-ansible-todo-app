# minikube-ansible-todo-app

Using Ansible playbook, set up a single Node Kubernetes cluster into a Virtual Machine to deploy the full stack of the complete "TODO" application provided by Docker in the following tutorial:
https://docs.docker.com/get-started/02_our_app/

Let's break down the configuration files in this repository starting with the deployment files:

Kubernetes Deployment for Todo Application
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
