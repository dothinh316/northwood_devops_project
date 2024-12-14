# northwood_devops_project
![K3 Architecture Design](k3_architecture.pdf "arch")

## **Overview**
This doc provides workflow to deploy a K3s Cluster on a Multipass Instance with infrastructure components and a Flask app with Celery Workers.  This workflow will only work if you are on a ARM64 architecture.  I tried to develop for AMD64 but that got too complex with the time I had.

## **Components**

### **1. Flask Application**
- A Python-based web application that serves requests and triggers Celery tasks.
- Exposed via an NGINX Ingress for external access.

### **2. Celery Workers**
- Background workers to process long-running tasks from the Flask app.
- Connected to a Redis broker for task queuing.

### **3. Redis**
- Acts as the message broker and result backend for Celery.

### **4. Celery Exporter**
- Exposes Prometheus metrics for Celery task events such as task runtime, successes, and failures.

### **5. NGINX Ingress Controller**
- Routes external traffic to the Flask app based on ingress rules.
- Configured to expose the application using a NodePort service or hostname.

### **6. Prometheus**
- Scrapes metrics from the Celery Exporter for monitoring and analysis.

### **7. Grafana**
- Visualizes metrics from Prometheus, including dashboards for Celery task performance.

## **Prerequisites**

1. **kubectl**
   - Required to manage and connect to the Kubernetes cluster.
   - [Install kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)

2. **Multipass**
   - Required to deploy a local Multipass instance for the Kubernetes cluster.
   - [Install Multipass](https://multipass.run/)

3. **Helm**
   - Required to deploy Helm charts from your local machine.
   - [Install Helm](https://helm.sh/docs/intro/install/)

4. **Docker**
   - Required to build and push Docker images for your components.
   - [Install Docker](https://docs.docker.com/get-docker/)

5. **ARM64 Machine**
   - The infrastructure and Docker images are designed and tested for ARM64 architecture.
   - Support for AMD64 was omitted due to project complexity.

---
## **Make Celery Container Deployment**
Python code for Flask app and Celery worker lives in [make_celery](https://github.com/dothinh316/make_celery). Please follow README.md instructions for build and deployment.  Currently there is a Github Actions workflow there to automate the build process from CI based on commit to `main` or a version tag

## **Design**
The Flask application serves two key endpoints:
1. `/` - Home endpoint
2. `/long_task` - Trigger a long running task

It is intended when a user hits `http://<Node-IP>:30080/long_task`, this will add a task to the broker, Redis. Then the Celery Worker will go and pick up that task to process.

## **Workflow**
1. Install Mutipass Instance - [multipass-k3s-setup](https://github.com/dothinh316/multipass-k3s-setup)
2. K3s deploys and binds the cluster to localhost only so you can only access the cluster from your Multipass VM.  Lets update the configs
so we expose the Cluster API through Multipass VM IP
    a. vim /etc/systemd/system/k3s.service
    b. Find the line with ExecStart and add the --bind-address and --advertise-address flags:
        example: ExecStart=/usr/local/bin/k3s server \
                    --bind-address=0.0.0.0 \
                    --advertise-address=192.168.64.2 \
                    --disable=traefik
    c.  sudo systemctl daemon-reload
        sudo systemctl restart k3s
    d. scp ubuntu@<VM's IP>:/etc/rancher/k3s/k3s.yaml ~/k3s.yaml
    e. vim ~/k3s.yaml 
        i. FIND AND REPLACE VM's IP
                clusters:
                    - cluster:
                        server: https://<VM's IP>:6443
                        certificate-authority-data: <...>
    f. export KUBECONFIG=~/k3s.yaml
3. Deployments - Deploy Observability Pipeline with Helm From Local Machine
    a. Clone [deployments](https://github.com/dothinh316/deployments)
    b. cd into deployments repo and install required helm charts with the values file provided
    c. Grafana
        * helm repo add grafana https://grafana.github.io/helm-charts
           helm repo update
        * kubectl create namespace grafana
        * helm install grafana grafana/grafana --namespace grafana -f grafana-values.yaml
        * Connect to Grafana from your local machine http://<MULTIPASS_IP>:32000
    d. Prometheus
        * helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
          helm repo update
        * kubectl create namespace prometheus
        * helm install prometheus prometheus-community/prometheus --namespace prometheus -f prometheus-values.yaml
    e. Redis
        * helm repo add bitnami https://charts.bitnami.com/bitnami
          helm repo update
        * kubectl create namespace redis
        * helm install redis bitnami/redis --namespace redis -f redis-values.yaml
    f. Nginx
        * helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
        * helm repo update
        * kubectl create namespace ingress-nginx
        * helm install nginx-ingress ingress-nginx/ingress-nginx
    g. Celery
        * cd celery_helm
        * helm install celery -f values.yaml .
4. Dashboards for Deployments - Follow [this](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/import-dashboards/) to add pre build dashboards
    a. https://grafana.com/grafana/dashboards/15282-k8s-rke-cluster-monitoring/
    b. https://grafana.com/grafana/dashboards/17508-celery-tasks-by-task/
    c. https://grafana.com/grafana/dashboards/763-redis-dashboard-for-prometheus-redis-exporter-1-x/
