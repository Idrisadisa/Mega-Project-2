GoldenCat Bank – End-to-End DevSecOps on AWS EKS

This project implements a complete DevSecOps pipeline for a sample banking application running on AWS EKS.

It covers:

- Infrastructure & platform setup on AWS using Terraform 
- CI pipeline (build, test, code quality, security scans, artifact & image publishing)
- CD pipeline to EKS with Ingress and TLS (Let’s Encrypt via `cert-manager`)
- Monitoring & observability with Prometheus and Grafana

Architecture Overview

High-Level Design

The platform runs on a single EKS cluster with multiple namespaces:

- `jenkins-namespace` – Jenkins controller & agents
- `nexus3-namespace` – Nexus3 artifact repository
- `sonar-namespace` – SonarQube for static analysis
- `webapps` – GoldenCat Bank application (bankapp + MySQL)
- `monitoring` – Prometheus & Grafana stack

An infra server (EC2) is used as the main control machine, with:

- Terraform
- kubectl
- Helm
- AWS CLI
- eksctl

The cluster runs on three worker nodes plus dedicated instances for Jenkins, SonarQube and Nexus:

EKS cluster overview:

---

CI Pipeline – `Mega-Project-CI`



CI Architecture

The CI pipeline is responsible for:

- Compiling and testing the Java / Spring Boot bank application
- Running static analysis and quality gates
- Scanning the filesystem and container image with Trivy
- Publishing artifacts to Nexus and container images to Docker Hub
- Triggering notifications and preparing the CD stage

Tools & Services

- Jenkins – Declarative pipeline
- GitHub – Source repository and webhooks
- Maven – Build and test
- SonarQube – Static code analysis & Quality Gate
- Trivy – Filesystem and Docker image scanning
- Nexus3 – Maven snapshot repository
- Docker Hub – Image registry
- Email – Pipeline result notification

Pipeline Stages

The Jenkins CI pipeline runs the following stages:

1. Tool Install – Ensure required tools (Maven, Trivy, etc.) are available.
2. Git Checkout – Clone the `main` branch of `Mega-Project-2`.
3. Compile – Compile the code with Maven.
4. Testing– Run unit tests.
5. Trivy_FS_Scan – Scan the Jenkins workspace for vulnerabilities.
6. Sonar Analysis – Push code metrics to SonarQube project `gcbank`.
7. Quality Gate Check – Wait on SonarQube’s Quality Gate result.
8. Build – Package the application JAR.
9. Publish To Nexus – Deploy artifacts to Nexus snapshot repository.
10. Docker Image Build & Scan Image – Build the Docker image and scan it with Trivy.
11. Push Docker Image – Push the image to Docker Hub.
12. Update Manifest File in Mega-Project-CD – Bump the Kubernetes image tag (e.g. `v67`).
13. Post Actions – Send email notification on success/failure.

#### Jenkins CI Graph




Artifacts & Nexus

Jenkins stores the compiled artifacts, while Nexus holds the Maven snapshots:

SonarQube Quality Dashboard

Static analysis and Quality Gate result for the `gcbank` project:

Email Notification

On successful pipeline completion, an email is sent summarizing the status and linking to the Jenkins console output:

---
CD Pipeline – `Mega-Project-CD`

### 3.1 CD Architecture

The CD pipeline deploys the latest Docker image to the **`webapps`** namespace in EKS and configures Ingress + TLS.





Tools & Services

- Jenkins – CD pipeline (`Mega-Project-CD`)
- Kubernetes plugin – `withKubeConfig` using a **service account** and kubeconfig secret
- AWS EKS – Target cluster (`powerdevops-cluster`)
- NGINX Ingress Controller – L7 routing into the cluster
- cert-manager – Automated certificate management
- Let’s Encrypt – Public TLS certificates

Kubernetes Resources

Within namespace `webapps`, the pipeline applies:
- `mysql` Deployment + Service + PVC using `ebs-sc`
- `bankapp` Deployment + Service
- HorizontalPodAutoscaler (HPA) for `bankapp`
- Ingress `bankapp-ingress`
- `ClusterIssuer` and Certificate resources for TLS

The main manifest apply step:

```bash
kubectl apply -n webapps -f Manifest/manifest.yaml
kubectl apply -n webapps -f Manifest/HPA.yaml
kubectl apply -n webapps -f Manifest/ci.yaml
kubectl apply -n webapps -f Manifest/ingress.yaml

Monitoring Stack – Prometheus & Grafana

Monitoring is installed via Helm using kube-prometheus-stack:

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  -f values.yaml \
  -n monitoring --create-namespace

The GoldenCat Bank application is exposed over HTTPS via NGINX Ingress and Let’s Encrypt TLS:
https://www.hkhpros.xyz 


