# 🎬 Netflix Clone – End-to-End DevSecOps CI/CD Pipeline

A production-style DevSecOps pipeline that builds, secures, and deploys a **Netflix Clone** application to **AWS EKS**, with full observability via **Prometheus** and **Grafana**.

The pipeline integrates **Jenkins, SonarQube, Trivy, OWASP Dependency-Check, Docker, Terraform, and Kubernetes** to demonstrate a real-world CI/CD + DevSecOps + Monitoring workflow.

---

## 🧰 Tech Stack

| Category            | Tools / Services                                   |
|----------------------|-----------------------------------------------------|
| CI/CD                | Jenkins (Declarative Pipeline)                       |
| Infra as Code         | Terraform (AWS EKS provisioning)                     |
| Cloud                 | AWS (EC2, EKS, S3, IAM, ELB, Security Groups)        |
| Containerization       | Docker, Docker Hub                                   |
| Orchestration          | Kubernetes (AWS EKS)                                 |
| Code Quality           | SonarQube                                            |
| Security Scanning      | Trivy (FS & Image Scan), OWASP Dependency-Check        |
| Monitoring             | Prometheus, Node Exporter, Grafana                    |
| Notifications           | Jenkins Email Extension (Gmail SMTP)                  |
| Data Source             | TMDB API (movie data)                                 |

---

## 🏗️ Architecture Overview

1. Code is pushed to GitHub.
2. Jenkins pipeline pulls the code, runs a **SonarQube** static code analysis + quality gate.
3. **OWASP Dependency-Check** and **Trivy (filesystem scan)** scan for vulnerable dependencies.
4. Docker image is built, tagged, and pushed to **Docker Hub**.
5. **Trivy** scans the built image for vulnerabilities.
6. The application is deployed to an **AWS EKS** cluster (provisioned via a separate Terraform pipeline) using `kubectl apply`.
7. **Prometheus + Node Exporter** scrape metrics from Jenkins and the servers; **Grafana** visualizes them on dashboards.
8. Jenkins sends build status emails at the end of every run.

---

## 📋 Prerequisites

- AWS account with an IAM role/user that has admin (or scoped EKS/EC2/S3) permissions
- A **TMDB API key** ([themoviedb.org](https://www.themoviedb.org/) → Settings → API)
- A Docker Hub account
- A Gmail account with an **App Password** (for Jenkins email notifications)
- Domain/DNS not required — accessed via public IP / LoadBalancer

> ⚠️ **Security note:** Never commit real API keys, SMTP passwords, or SonarQube tokens to this repo. Store them as **Jenkins Credentials** and reference them by ID, as shown in the pipeline stages below.

---

## 🚀 Setup Walkthrough

### Step 1 – Provision the Jenkins/Build EC2 Instance
Launch an **Ubuntu 22/24.04, t2.large, 30GB** EC2 instance and attach an IAM role with the required permissions.

Install core tooling via script:
- **Terraform**
- **kubectl**
- **AWS CLI**

### Step 2 – Install Jenkins, Docker, Trivy & SonarQube
- Install **Java (Temurin 17)** and **Jenkins**, start the Jenkins service (`:8080`)
- Install **Docker**, add the `jenkins`/`ubuntu` user to the `docker` group
- Install **Trivy** for vulnerability scanning
- Run **SonarQube** as a Docker container (`:9000`)

### Step 3 – TMDB API Key
Sign up on TMDB → Settings → API → create a developer application to get an API key used at Docker build time (`TMDB_V3_API_KEY`).

### Step 4 – Monitoring Stack (separate EC2 – t2.medium)
- Install **Prometheus** (`:9090`) as a systemd service
- Install **Node Exporter** (`:9100`) and register it as a Prometheus scrape target
- Install **Grafana** (`:3000`), connect it to Prometheus as a data source, and import dashboard **ID 1860** (Node Exporter)

### Step 5 – Jenkins ↔ Prometheus Integration
- Install the **Prometheus plugin** in Jenkins
- Add Jenkins as a scrape target (`/prometheus` metrics path) in `prometheus.yml`
- Import Grafana dashboard **ID 9964** (Jenkins metrics)

### Step 6 – Email Notifications
- Install the **Email Extension** plugin
- Configure Gmail SMTP (`smtp.gmail.com:465`, SSL) using a Gmail **App Password** stored in Jenkins Credentials
- Set default triggers: `Always`, `Failure - Any`

### Step 7 – Jenkins Tooling & Plugins
Install and configure:
- Eclipse Temurin Installer (JDK)
- SonarQube Scanner plugin
- NodeJS plugin
- Docker / Docker Pipeline / Docker Commons / Docker API plugins

Configure JDK, SonarQube Scanner, and NodeJS tool auto-installers under **Manage Jenkins → Tools**.

Connect SonarQube ↔ Jenkins via a **Secret Text** credential (`sonar-token`) and a webhook back to Jenkins (`/sonarqube-webhook/`).

### Step 8 – CI Pipeline (Build & Quality Gate)
A **Declarative Pipeline** that:
1. Cleans workspace & checks out from GitHub
2. Runs SonarQube analysis + quality gate
3. Installs project dependencies (`npm install`)

### Step 9 – Security Scans
- **OWASP Dependency-Check** plugin installed and integrated into the pipeline
- **Trivy filesystem scan** (`trivy fs .`) run before the Docker build

### Step 10 – Docker Build, Scan & Push
- Docker Hub credentials added to Jenkins (`ID: docker`)
- Pipeline builds the image with the TMDB API key as a build arg, tags it, and pushes to Docker Hub
- **Trivy image scan** run against the pushed image
- Scan reports (`trivyfs.txt`, `trivyimage.txt`) attached to the build email

### Step 11 – EKS Cluster Provisioning (Terraform, via Jenkins)
A separate **parameterized pipeline** (`action`: `apply` / `destroy`) that:
1. Checks out the Terraform config (`EKS_TERRAFORM/` directory)
2. Runs `terraform init → validate → plan → apply/destroy`
3. Provisions the AWS EKS cluster (state stored in a private S3 bucket)

After provisioning:
```
aws eks update-kubeconfig --region <region> --name <cluster-name>
kubectl get nodes
```

### Step 12 – Kubernetes Deployment
- Kubernetes plugin + credentials (kubeconfig saved as a Jenkins **Secret File**, `ID: k8s`) added
- Final pipeline adds a **Deploy to Kubernetes** stage:
  ```
  kubectl apply -f deployment.yml
  kubectl apply -f service.yml
  ```
- Manifests live under the `Kubernetes/` directory in this repo

### Step 13 – Access the Application
Retrieve the LoadBalancer/service URL:
```
kubectl get svc
```
Open the returned endpoint in your browser to view the deployed Netflix Clone.

### Step 14 – Teardown
To avoid ongoing AWS charges:
```
# Destroy the EKS cluster via the Terraform Jenkins pipeline (action = destroy)
```
Then manually:
- Terminate the Jenkins/build EC2 instance
- Terminate the Prometheus/Grafana EC2 instance
- Delete any leftover Load Balancers and Security Groups

---

## 📊 Monitoring Dashboards

| Dashboard      | Source              | Dashboard ID |
|-----------------|----------------------|--------------|
| Node Exporter   | Server-level metrics | `1860`       |
| Jenkins         | CI/CD pipeline metrics | `9964`     |

---

## 📁 Repository Structure

```
.
├── EKS_TERRAFORM/         # Terraform code to provision the AWS EKS cluster
├── Kubernetes/            # deployment.yml & service.yml manifests
├── netflix_deployment_proof/   # 📸 Screenshots/proof of the working deployment 
├── Dockerfile
└── README.md
```

> 📌 **Note:** The `netflix_deployment_proof` folder is contain proof of the working setup (Jenkins pipeline runs, SonarQube reports, Trivy scan outputs, Grafana dashboards, and the live app) .

---

## 🔐 Credentials Used in Jenkins (reference only — configure your own)

| Credential ID   | Kind                 | Purpose                          |
|-------------------|----------------------|-----------------------------------|
| `docker`          | Username & Password  | Docker Hub push access             |
| `sonar-token`      | Secret Text           | SonarQube authentication            |
| `mail`             | Username & Password  | Gmail SMTP for build notifications  |
| `k8s`              | Secret File           | Kubeconfig for EKS deployment        |

---

## 🙋 Author

**Mayur Patil**
DevOps & Cloud Enthusiast 

GitHub: [@mayurpatil0708](https://github.com/mayurpatil0708)

---

## 📄 License

This project is intended for learning and portfolio purposes.
