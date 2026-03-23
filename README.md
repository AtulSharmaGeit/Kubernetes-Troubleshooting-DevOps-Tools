# Kubernetes Troubleshooting & DevOps Tools

## 📌 Overview
This project simulates a **client-support environment** for DevOps platforms running on Kubernetes.  
It focuses on **incident resolution, root cause analysis, and preventive measures** for tools commonly used in CI/CD workflows: GitLab, Jenkins, SonarQube, Nexus, and Grafana.  

The goal is to practice **real-world troubleshooting scenarios** and document solutions in a knowledge base format.

---

## ⚙️ Tech Stack
- **Kubernetes** (Minikube / kind for local cluster simulation)
- **Helm** (package manager for Kubernetes)
- **DevOps Tools**: GitLab, Jenkins, SonarQube, Nexus, Grafana
- **Monitoring**: Grafana dashboards
- **Version Control**: GitHub (for repo + documentation)

---

## 🚀 Setup Instructions

**🌐Phase 1: Environment Setup**

1. **Install Kubernetes CLI (`kubectl`)**<br>
    `kubectl` is the command-line tool to interact with Kubernetes clusters.

    ```bash
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

    chmod +x kubectl

    sudo mv kubectl /usr/local/bin/

    kubectl version --client
    ```

2. **Install Minikube**<br>
    Minikube creates a local Kubernetes cluster.

    ```bash
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

    sudo install minikube-linux-amd64 /usr/local/bin/minikube
    ```

3. **Start Kubernetes Cluster with Minikube**<br>
    Now we can start our cluster with enough resources for DevOps tools:

    ```bash
    minikube start --memory=4096 --cpus=2
    ```
    - `--memory=4096` - Allocates 4 GB RAM.

    - `--cpus=2` - Allocates 2 CPU cores.

    Verify cluster is running:

    ```bash
    kubectl get nodes
    ```
    We should see one node (the Minikube VM).

4. **Install Helm (Package Manager for Kubernetes)**
    Helm helps us install apps like Jenkins, SonarQube, Grafana easily.

    ```bash
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

    helm version
    ```

**🌐Phase 2: Deploy DevOps Tools on Kubernetes**

1. **Deploy Jenkins on Kubernetes**<br>
    Add Jenkins Helm Repository - Helm uses repositories to fetch charts (pre-packaged Kubernetes apps).

    ```bash
    helm repo add jenkins https://charts.jenkins.io

    helm repo update
    ```
    Deploy Jenkins into our cluster:
    ```bash
    helm install jenkins jenkins/jenkins
    ```
    - This will create a Jenkins deployment, service, and related pods.

    Check if Jenkins pods are running:
    ```bash
    kubectl get pods
    ```
    We should see something like:
    ```Code
    NAME                         READY   STATUS    RESTARTS   AGE
    jenkins-xxxxxx               1/1     Running   0          2m
    ```
    Since Minikube runs locally, we can use **port-forwarding**:
    ```bash
    kubectl port-forward svc/jenkins 8080:8080
    ```
    - Now open browser at `http://localhost:8080`
    
    Jenkins generates a default admin password stored in a Kubernetes secret.
    ```bash
    kubectl exec --namespace default -it $(kubectl get pod -l app.kubernetes.io/component=jenkins-controller -o jsonpath="{.items[0].metadata.name}") -- cat /run/secrets/additional/chart-admin-password
    ```
    - Copy this password and use it to log in as **admin**.

    Create a simple freestyle job.

    Run it to confirm Jenkins is working.

    If it fails, check logs:
    ```bash
    kubectl logs <jenkins-pod>
    ```

2. **Deploy SonarQube**<br>
    Add Helm Repo
    ```bash
    helm repo add oteemocharts https://oteemo.github.io/charts

    helm repo update
    ```
    Install SonarQube
    ```bash
    helm install sonarqube oteemocharts/sonarqube
    ```
    Verify Pod
    ```bash
    kubectl get pods
    ```
    - Look for `sonarqube-xxxx` pod in **Running** state.
    
    Port‑forward service:
    ```bash
    kubectl port-forward svc/sonarqube 9000:9000
    ```
    Open browser - `http://localhost:9000`

    Default login:
    - Username: `admin`
    - Password: `admin`

3. **Deploy Nexus Repository Manager**<br>
    Add Helm Repo
    ```bash
    helm repo add sonatype https://sonatype.github.io/helm3-charts/

    helm repo update
    ```
    Install Nexus
    ```bash
    helm install nexus sonatype/nexus-repository-manager
    ```
    Verify Pod
    ```bash
    kubectl get pods
    ```
    Port‑forward service:
    ```bash
    kubectl port-forward svc/nexus-repository-manager 8081:8081
    ```
    - Open browser - `http://localhost:8081`

    Default login:
    - Username: `admin`
    - Password: `admin123`

4. **Deploy Grafana**<br>
    Add Helm Repo
    ```bash
    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update
    ```
    Install Grafana
    ```bash
    helm install grafana grafana/grafana
    ```
    Verify Pod
    ```bash
    kubectl get pods
    ```
    Port‑forward service:
    ```bash
    kubectl port-forward svc/grafana 3000:3000
    ```
    - Open browser - `http://localhost:3000`

    Default login:
    - Username: `admin`
    - Password: `admin` (we’ll be prompted to change it).

---

## ⚙️ Simulating Incidents & Troubleshooting

1. **SonarQube OOM (Out Of Memory) Error**

    Goal: Practice diagnosing pod crashes due to resource limits.<br>
    Steps:
    - Edit SonarQube deployment to set very low memory limits:
        ```bash
        kubectl edit deployment sonarqube
        ```
        - Under `resources.limits.memory`, set something small like `128Mi`.

    - Watch pod crash:
        ```bash
        kubectl get pods

        kubectl logs <sonarqube-pod>
        ```
        - We’ll see `OOMKilled`.
    - **Fix:** Increase memory limits (e.g., `1024Mi`) and redeploy.
    - **Preventive Action:** Set realistic resource requests/limits in YAML.

2. **GitLab SSH Cloning Issue**

    Goal: Simulate repo access problems.<br>
    Steps:
    - Try cloning with wrong SSH key:
        ```bash
        git clone git@gitlab.example.com:project/repo.git
        ```
        - It fails.
    - Debug:
        - Test SSH connection:
            ```bash
            ssh -T git@gitlab.example.com
            ```
        - Check ingress rules:
            ```bash
            kubectl describe ingress gitlab
            ```
        - Verify network connectivity.
    - **Fix:** Correct SSH keys, ingress rules, or firewall settings.
    - **Preventive Action:** Document SSH setup and ingress configs.

3. **Jenkins Agent Failure**

    Goal: Practice job failures due to missing agents.<br>
    Steps:
    - Scale agents down to 0:
        ```bash
        kubectl scale deployment jenkins-agent --replicas=0
        ```
    - Run a job ---> it fails (no agents available).
    - Debug:
        ```bash
        kubectl get pods
        kubectl describe pod <jenkins-pod>
        ```
    - **Fix:** Scale agents back up, check node resources.
    - **Preventive Action:** Add monitoring for agent availability.

4. **Grafana Datasource Error**

    Goal: Practice dashboard troubleshooting.<br>
    Steps:
    - Misconfigure Prometheus datasource in Grafana (wrong URL).
    - Dashboard shows “No data” or errors.
    - Debug:
        - Check Grafana logs:
            ```bash
            kubectl logs <grafana-pod>
            ```
        - Verify datasource settings in Grafana UI.
        - Test Prometheus endpoint manually:
            ```bash
            curl http://prometheus:9090/metrics
            ```
    - **Fix:** Correct datasource URL and credentials.
    - **Preventive Action:** Document datasource setup and test connectivity before dashboards.
