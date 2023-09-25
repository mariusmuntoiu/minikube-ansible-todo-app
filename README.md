# minikube-ansible-todo-app

Using Ansible playbook, set up a single Node Kubernetes cluster into a Virtual Machine to deploy the full stack of the complete "TODO" application provided by Docker in the following tutorial:
https://docs.docker.com/get-started/02_our_app/

In order to set up a single Node Kubernetes cluster we can use **Minikube**:

Minikube is an open-source tool that allows us to run Kubernetes (K8s) locally. It sets up a single-node Kubernetes cluster on our personal computer so we can test, develop, and experiment with Kubernetes applications without needing a larger, potentially costly, cluster.


Let's break down the configuration files in this repository starting with the deployment files:

**Minikube Deployment for Todo Application:**

This document outlines the Kubernetes resources used to deploy a Todo Application and its supporting infrastructure.

**Overview**


The application consists of two main components:


**Redis** - An in-memory data structure store, used as a database, cache, and message broker.

**Todo-app** - A web application to manage todos which communicates with Redis for data storage.





The Kubernetes resources utilized to deploy these components are:


**PersistentVolume (PV) & PersistentVolumeClaim (PVC)**


**Deployments**


**Services**


**Ingress**


I will try to breakdown each one starting with:



**PersistentVolume (PV) & PersistentVolumeClaim (PVC)**


**redis-pv**


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

**redis-pvc**


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

**Deployments:**


**Redis Deployment**


Deploys a single replica of the redis:alpine image. This Redis instance uses the storage provisioned by the redis-pvc.


**Key Configurations:**


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

**Todo-app Deployment**


Deploys a single replica of the mmarius19/getting-started:latest image. This is the Todo web application.


**Key Configurations:**


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

**Services**


**Redis Service**


A ClusterIP service (default type) that provides internal connectivity to the Redis deployment within the Kubernetes cluster.

```yaml
apiVersion: v1
kind: Service
...
spec:
  ports:
    - port: 6379
```


**Todo-app Service**


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


**Ingress**


**todo-ingress**


An Ingress resource that provides HTTP routing to the Todo web application. It allows external users to access the app using the hostname mariusmuntoiu.com


**Key Configurations:**


Path: The ingress routes traffic for the path / to the todo-app service on port 3000.
Ingress resource is designed to route external traffic coming to "mariusmuntoiu.com" on any path starting with / to the todo-app service on port 3000 in the Kubernetes cluster. This is particularly useful when you have a domain that you want to point to a specific service within your minikube cluster, and that service is listening on a specific port.


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
...
spec:
  rules:
    - host: mariusmuntoiu.com
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

**NOTE:**
I've divided the main deployment file **to-do-app-full-deploy.yaml** for better readability in order to not overpopulate the Ansible Playbook with tasks for kubectl apply on every yaml file.


This setup ensures data persistence for Redis, scalable deployment for the web application, internal communication via services, and external access via a LoadBalancer and Ingress. 
Remember to map the domain ("your-domain.com") to the appropriate IP address in your environment (e.g., /etc/hosts or DNS) for external access.

Now let's break down the Ansible Playbook


**We are assuming that the target virtual machine only has a basic Linux installation. Packages that are specific to the
deployment of this application are not already available and must be installed.**


**NOTE**: We must check if the target VM has an ansible user configured. If not, we must configure it and create a SSH key so that we can communicate with our Control Node passwordless.


```bash
#For Ubuntu
sudo useradd ansible
sudo usermod -aG sudo ansible
#create a .ssh directory for the ansible user
cd /home/ansible/
mkdir .ssh
cd .ssh
ssh-keygen
```

On our Control Node use ssh-copy-id to copy the key from the worker node.

```bash

ssh-copy-id ansible@worker_node_IP

```

This Ansible playbook sets up Minikube, Docker, installs essential libraries/dependencies and deploys a Todo Application on to a target Virtual Machine.


**1. Ensure software-properties-common is installed:**
This package is required to add external repositories using the apt-add-repository command.

```yml
- name: Ensure software-properties-common is installed (for apt-add-repository)
  apt:
    name: software-properties-common
    state: present
```

**2. Ensure python3-apt is installed:**
The package python3-apt provides a high-level interface for the APT library, making it easier to manage packages.

```yml
- name: Ensure python3-apt is installed
  apt:
    name: python3-apt
    state: present
```

**3. Install python3-pip:**
Installs the Python3 version of pip, the Python package installer.

```yml
- name: Install python3-pip
  apt:
    name: python3-pip
    state: present
```

**4. Install required Python modules:**
Installs the Python modules openshift and kubernetes required for Kubernetes operations.

```yml
- name: Install required Python modules
  pip:
    name:
     - openshift
     - kubernetes
    executable: pip3
```

**5. Install Minikube dependencies:**
Ensures the installation of packages required by Minikube.

```yml
- name: Install Minikube dependencies
  apt:
    name:
      - apt-transport-https
      - curl
    state: present
```

**6. Add Google's apt key:**
Adds Google's APT key, a requirement for adding Google's repositories.

```yml
- name: Add Google's apt key
  apt_key:
    url: "https://packages.cloud.google.com/apt/doc/apt-key.gpg"
    state: present
```
**7. Add Kubernetes apt repository:**
Adds the Kubernetes APT repository for installing kubeadm, kubectl, and kubelet.

```yml
- name: Add Kubernetes apt repository
  apt_repository:
    repo: deb https://apt.kubernetes.io/ kubernetes-xenial main
    state: present
```

**8. Install kubelet, kubeadm, and kubectl:**
These are Kubernetes components: kubelet (node agent), kubeadm (for cluster management), and kubectl (command line tool).

```yml
- name: Install kubelet kubeadm kubectl
  apt:
    name:
      - kubelet
      - kubeadm
      - kubectl
    state: present
```

**9. Install Docker dependencies:**
Ensures the installation of packages required by Docker.

```yml
- name: Install Docker dependencies
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - software-properties-common
    state: present
```

**10. Update APT cache:**
Refreshes the local package database, ensuring the latest package information is used.

```yml
- name: Update APT cache
  apt:
    update_cache: yes
```

**11. Add Docker's official GPG key:**
Adds Docker's official GPG key, which ensures the packages installed from Docker's repository are authenticated.

```yml
- name: Add Docker's official GPG key
  apt_key:
    url: "https://download.docker.com/linux/ubuntu/gpg"
    state: present
```

**12. Add Docker repository:**
This task adds Docker's official repository, which provides the latest versions of Docker.

```yml
- name: Add Docker repository
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
    state: present
```

**13. Install Docker CE:**
Installs the Docker Community Edition along with its CLI and the containerd runtime.

```yml
- name: Install Docker CE
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present
```

**14. Ensure Docker is started and enabled at boot:**
Makes sure the Docker service is running and set to start on system boot.

```yml
- name: Ensure Docker is started and enabled at boot
  systemd:
    name: docker
    state: started
    enabled: yes
```

**15. Install Minikube:**
Downloads and installs the Minikube binary, which is used to run a single-node Kubernetes cluster.

```yml
- name: Install Minikube
  get_url:
    url: https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    dest: /usr/local/bin/minikube
    mode: '0755'
```

**16. Add user to docker group:**
This gives the user vmuser permissions to run Docker commands without sudo.

```yml
- name: Add user to docker group
  user:
    name: dockervm
    groups: docker
    append: yes
```

**17. Start Minikube:**
Starts a Minikube cluster using the Docker driver. The command is run as the dockervm user.

```yml
- name: Start Minikube
  command: minikube start --driver=docker
  become: no
  become_user: dockervm
```

**18.Enable Ingress Addon in Minikube:**
Activates the ingress controller for Minikube.


```yml
- name: Enable Ingress Addon in Minikube
  command: minikube addons enable ingress
  become: no
```


 **19.Check if Helm is already installed:**
 Checks for the presence of Helm.

 
```yml
- name: Check if Helm is already installed
  command: helm version --short
  register: helm_version
  failed_when: false
  changed_when: false
```

**20.Download Helm binary:**
 Downloads Helm if not installed
 

```yml
- name: Download Helm binary
  get_url:
    url: "https://get.helm.sh/helm-v3.7.0-linux-amd64.tar.gz" 
    dest: "/tmp/helm-v3.7.0-linux-amd64.tar.gz"
    mode: '0755'
  when: helm_version.rc != 0
 ```

**21.Extract Helm tarball:**
Extracts the Helm binary.


```yml
- name: Extract Helm tarball
  unarchive:
    src: "/tmp/helm-v3.7.0-linux-amd64.tar.gz"
    dest: "/tmp"
    remote_src: true
  when: helm_version.rc != 0
 ```

**22.Move Helm binary to PATH:**
Makes Helm available system-wide.


```yml
- name: Move Helm binary to PATH
  command: mv /tmp/linux-amd64/helm /usr/local/bin/helm
  when: helm_version.rc != 0
```

**23. Verify Helm Installation:**
Confirms Helm is properly installed.


```yml
- name: Verify Helm Installation
  command: helm version --short
  register: post_install_helm_version
  failed_when: post_install_helm_version.rc != 0
```

**24.Ensure Helm repo for nginx-ingress is added:**
Adds the nginx-ingress Helm repository.


```yml
- name: Ensure Helm repo for nginx-ingress is added
  command: helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  register: repo_added
  changed_when: "'\"ingress-nginx\" has been added to your repositories' in repo_added.stdout"
```

**25.Check if nginx-ingress is already installed:**
Checks for the presence of the nginx-ingress Helm chart.


```yml
- name: Check if nginx-ingress is already installed
  command: helm list -n ingress-nginx | grep ingress-nginx
  register: nginx_ingress_installed
  failed_when: false
  changed_when: false
```

**26.Update Helm repo:**
Updates the Helm repositories.


```yml
- name: Update Helm repo
  command: helm repo update
```

**27. Deploy Todo Application:**
Deploys a Todo Application to the Minikube cluster. This task applies the Kubernetes manifest located at /home/ansible/deployments-minikube/todo-app-full-deploy.yaml.

```yml
 - name: Deploy TODO Application
      community.kubernetes.k8s:
        kubeconfig: /home/ansible/.kube/config
        context: minikube
        namespace: default
        apply: yes
        src: /home/ansible/deployments-minikube/todo-app-full-deploy.yaml
``` 

**28.Install Nginx:**
Installs the Nginx web server.


```yml
- name: Install Nginx
  apt:
    name: nginx
    state: present
```

**29.Get Minikube IP:**
Retrieves the IP address of the Minikube cluster.


```yml
- name: Get Minikube IP
  command: minikube ip
  register: minikube_ip
```


**30.Get Service NodePort:**
Fetches the NodePort of the Todo application's service.


```yml
- name: Get Service NodePort
  command: kubectl get svc todo-app-service -o=jsonpath='{.spec.ports[0].nodePort}'
  register: service_nodeport
```


**31.Create Nginx reverse proxy configuration:**
Sets up Nginx to forward traffic to the Todo application.


```yml
- name: Create Nginx reverse proxy configuration
  template:
    src: /path/to/nginx-todo-reverse-proxy.conf.j2
    dest: /etc/nginx/conf.d/todo-reverse-proxy.conf
```

**32.Enable Nginx configuration:**
Activates the newly created Nginx configuration.


```yml
- name: Enable Nginx configuration
  command: ln -s /etc/nginx/sites-available/todo-reverse-proxy /etc/nginx/sites-enabled/
```

**33.Test Nginx configuration:**
 Validates the Nginx configuration.


```yml
- name: Test Nginx configuration
  command: nginx -t
```

**34.Restart Nginx:**
Reloads Nginx to apply changes.


```yml
- name: Restart Nginx
  systemd:
    name: nginx
    state: restarted
```




**STATUS/TODO:**

Helm Charts with ansible-playbook.


Implement HTTPS/SSL - certbot Task in Ansible playbook - For the purpose of the presentation, at the moment HTTPS for the URL is configured on the server via certbot. 



**Conclusion:**
Application works as expected. You can access it at: https://mariusmuntoiu.com


Use incoginito in your preferred browser in case the url does not work.


Please feel free to review or contribute to my ansible-playbook exercise.
