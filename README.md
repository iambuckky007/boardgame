
Project Overview
============================================
This project showcases a complete end-to-end CI/CD pipeline on AWS using Jenkins, SonarQube, Nexus, Kubernetes (EKS), Trivy, Prometheus, and Grafana.
The pipeline automates builds, ensures security checks, and provides real-time monitoring of deployments into Kubernetes.
The pipeline ensures secure, automated, and monitored deployments into Kubernetes.


Technologies Used
==============================================
1. CI/CD: Jenkins

2. Code Quality: SonarQube

3. Artifact Repository: Nexus

4. Container Security: Trivy

5. Container Registry: AWS ECR

6. Orchestration: Kubernetes on AWS EKS

7.Monitoring & Visualization: Prometheus, Grafana
 
8. Cloud Infrastructure: AWS EC2, IAM, NLB



Architecture Overview
==================================================================

1. EC2 Instances
Main Web Server: To SSH into other server(such jenkins, nexus, etc.)
Jenkins Server (t3.medium): CI/CD Orchestration
SonarQube Server (t3.medium): Code Quality & Security Scanning
Nexus Server (t3.medium): Artifact Repository (JAR/TAR storage)
Monitoring Server (t3.medium): Prometheus & Grafana

2. AWS ECR
Used to store docker images after build and are pulled for deployment.

3. AWS EKS Cluster
Managed Kubernetes control plane
Worker Node Group (2 EC2 nodes)
Integrated with AWS ECR for storing container images

4. Monitoring Layer
Prometheus + Grafana dashboards
Metrics collected via node-exporter and kube-state-metrics

5. Jenkins Setup
Opened port 8080 in the security group to allow Jenkins web UI access.
Allowed SSH from single EC2(Main web server)
Installed essential Jenkins plugins. (sonar, nexus, docker, aws creds, maven, jdk etc)
Installed trivy on this EC2 server itself.

6. SonarQube Setup (Code Quality)
Deployed SonarQube on a separate EC2 instance (port 9000).
Allowed SSH from single EC2(Main web server)
Configured webhook → sends Quality Gate results back to Jenkins.

7. Nexus Repository Setup (Artifact Management)
Installed Nexus Repository Manager on an EC2 server (port 8081).
Allowed SSH from single EC2(Main web server)
Created dedicated Maven hosted repositories for JARs (application builds)

8. AWS EKS Cluster (Kubernetes Deployment)
Provisioned an EKS cluster using AWS console/CLI.
Configured IAM roles & RBAC bindings for Jenkins to deploy workloads.
Added a managed node group with two EC2 worker nodes. On Jenkins EC2:
Installed kubectl and AWS CLI.
Updated aws-auth ConfigMap to allow node → cluster communication.


9. Monitoring & Observability
Provisioned Prometheus & Grafana on a monitoring EC2 instance.
Open 9090 (Prometheus) and 3000 (Grafana) in the EC2 security group.
Allowed SSH from single EC2(Main web server)

Final Workflow
=======================================================
1. Developer pushes code → GitHub triggers Jenkins job.

2. Jenkins pipeline executes:

  -> Code analysis with SonarQube

  -> Quality Gate validation

  -> Build artifacts → Nexus

  -> Docker image build → Trivy scan

  -> If secure → Push to AWS ECR

3. Deploy to EKS via manifests

4. Application exposed via NLB.

5. Prometheus scrapes metrics → Grafana dashboards display insights.

6. Email notifications sent on success/failure.
