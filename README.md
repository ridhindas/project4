🚀 End-to-End DevSecOps & GitOps Pipeline for Django Application
A complete, production-grade CI/CD and GitOps pipeline for a Django-based Complaint Management System. This project demonstrates modern DevOps practices, including automated security scanning, containerization, and continuous deployment using ArgoCD.
📋 Table of Contents
Architecture Overview
Prerequisites
Step 1: Repository & Secrets Setup
Step 2: Local Kubernetes Cluster
Step 3: CI/CD Pipeline (GitHub Actions)
3.1 Pipeline Workflow
3.2 Security Scanning (DevSecOps)
3.3 Containerization & Registry
3.4 GitOps Manifest Update
3.5 Viewing Reports
Step 4: GitOps Deployment (ArgoCD)
Accessing the Application
Troubleshooting
Future Enhancements
🏗️ Architecture Overview
mermaid





Code
Preview
🛠️ Prerequisites
Ensure you have the following installed and configured on your local machine:
Git & GitHub Account
Docker (Desktop or Engine)
Minikube (for local Kubernetes cluster)
kubectl (Kubernetes command-line tool)
Docker Hub Account (for image registry)
GitHub Personal Access Token (PAT) with repo scope (for ArgoCD authentication)
🔐 Step 1: Repository & Secrets Setup
Before running the pipeline, configure the following Repository Secrets in GitHub (Settings > Secrets and variables > Actions):
Secret Name
Description
DOCKER_USERNAME
Your Docker Hub username (e.g., ridhindas)
DOCKER_PASSWORD
Your Docker Hub Access Token (Read & Write permissions)
PAT_TOKEN
GitHub Personal Access Token with repo scope (for GitOps auto-commit)
⚠️ Security Note: Never commit Kubernetes secrets (like ArgoCD repo credentials) to Git. GitHub's secret scanning will block it. Apply them directly to the cluster.
💻 Step 2: Local Kubernetes Cluster
Start Minikube (using Docker driver):
bash

1
Verify the cluster is running:
bash

1
⚙️ Step 3: CI/CD Pipeline (GitHub Actions)
The pipeline is defined in .github/workflows/pipeline.yml. On every push to main, it performs automated security scanning, builds the Docker image, and updates the Kubernetes manifests.
3.1 Pipeline Workflow
The pipeline consists of two jobs:
Job 1: security-and-build
This job runs on every push/PR to main and performs:
Step
Tool
Purpose
Checkout & Setup
actions/checkout@v3, actions/setup-python@v4
Clones repo and sets up Python 3.11 environment
SAST Scan
bandit
Scans Python code for security vulnerabilities (SQL injection, XSS, hardcoded secrets)
Dependency Scan
pip-audit
Checks requirements.txt for known CVEs in Python packages
Docker Build
docker build
Builds the Django application image with tag ridhindas/project4-django:<commit-sha>
Health Check
curl
Waits for Django to start (up to 60 seconds) before scanning
DAST Scan
OWASP ZAP
Dynamically scans the running application for runtime vulnerabilities
Docker Push
docker push
Pushes the image to Docker Hub with both commit SHA and latest tags
Job 2: update-gitops-repo
This job runs after security-and-build succeeds:
Updates k8s/deployment.yaml with the new Docker image tag
Commits and pushes the change back to GitHub (triggers ArgoCD sync)
3.2 Security Scanning (DevSecOps)
SAST (Static Application Security Testing)
Tool: Bandit
Bandit analyzes Python source code for common security issues:
Hardcoded passwords and API keys
SQL injection vulnerabilities
Insecure use of cryptographic functions
Unsafe YAML loading
Command:
bash

1
-r . : Recursively scan all Python files
-x ./venv,./__pycache__ : Exclude virtual environments and cache
-ll : Report only medium and high severity issues
Dependency Scanning
Tool: pip-audit
Checks all Python dependencies in requirements.txt against the OSV database for known vulnerabilities.
Command:
bash

1
DAST (Dynamic Application Security Testing)
Tool: OWASP ZAP
ZAP runs as a Docker container and actively scans the running Django application for:
Cross-Site Scripting (XSS)
SQL Injection
Missing security headers (CSP, X-Frame-Options, etc.)
Cookie security flags
Information disclosure
Configuration:
yaml

12345
3.3 Containerization & Registry
Dockerfile Structure
dockerfile

12345678910111213141516171819202122
Key Features:
Multi-stage build optimization
System dependencies for mysqlclient
Automatic database migration on startup
Non-root execution (best practice for production)
Docker Hub Push
The pipeline pushes two tags:
Commit SHA tag: ridhindas/project4-django:c069de850043fe37fa702bb8351603bad3b4e42a (immutable, for rollback)
Latest tag: ridhindas/project4-django:latest (mutable, for convenience)
3.4 GitOps Manifest Update
After a successful build, the pipeline automatically updates the Kubernetes deployment manifest:
yaml

1234567891011
This triggers ArgoCD to detect the change and sync the cluster automatically.
3.5 Viewing Reports
ZAP Security Reports (GitHub Artifacts)
The pipeline uploads three report formats as GitHub Artifacts:
report_html.html: Interactive HTML report with color-coded findings
report_json.json: Machine-readable JSON for integration with other tools
report_md.md: Markdown summary for quick review
How to download:
Go to GitHub → Actions tab
Click on a successful workflow run
Scroll to Artifacts section
Click zap-security-reports to download the ZIP file
GitHub Issues (Automated)
The ZAP action automatically creates (or updates) a GitHub Issue titled "ZAP Baseline Scan Results" with:
Summary of all warnings
Detailed findings per URL
Remediation recommendations
🔄 Step 4: GitOps Deployment (ArgoCD)
ArgoCD continuously monitors the k8s/ directory in this repository and syncs the state to the Kubernetes cluster.
4.1 Install ArgoCD
bash

12
4.2 Create ArgoCD AppProject (Permissions)
bash

123456789101112131415
4.3 Add GitHub Repository Credentials (Securely)
Replace YOUR_NEW_PAT_TOKEN with your actual GitHub PAT.
bash

12345678
4.4 Create the ArgoCD Application
bash

12345678910111213141516171819202122
🌐 Accessing the Application
Verify Deployment:
bash

12
Access via Minikube:
bash

1
Copy the output URL and paste it into your web browser.
Access ArgoCD UI:
bash

12345
Navigate to https://localhost:8081 (accept the self-signed cert warning) and log in with username admin and the retrieved password.
🔧 Troubleshooting
Issue
Solution
Port 8080/8081 already in use
Kill the existing process (lsof -ti:8081 | xargs kill) or use a different port (e.g., 8082:443).
ArgoCD "Authentication Required"
Your GitHub PAT expired or was revoked. Recreate the project4-repo-secret with a new token (see Step 4.3).
GitHub Push Rejected (Secret Scanning)
You accidentally committed a secret. Run git rm --cached <file>, add it to .gitignore, amend the commit, and push.
Django Pod CrashLoopBackOff
Check logs: kubectl logs <pod-name>. Usually caused by missing mysqlclient or unapplied migrations.
Bandit/pip-audit Failures
Fix the security issues in your code or update vulnerable dependencies, then push again.
ZAP Scan Fails (Connection Refused)
Django app crashed on startup. Check the "Wait for App to be Ready" step logs for the error.
🚀 Future Enhancements
To make this pipeline truly production-ready, the following enhancements are planned:
Persistent Storage (PV/PVC): To retain uploaded media files and database state across pod restarts.
Production Database: Migrate from SQLite to a managed PostgreSQL instance or a dedicated Kubernetes StatefulSet.
Ingress Controller: Replace NodePort with Nginx Ingress for clean, domain-based routing (e.g., app.project4.local).
Observability: Integrate Prometheus and Grafana for real-time cluster and application monitoring.
Image Vulnerability Scanning: Add Trivy or Snyk to scan Docker images for OS-level vulnerabilities.
Automated Rollback: Configure ArgoCD to automatically rollback if health checks fail after deployment.
📜 License
This project is open-source and available for educational and portfolio purposes.
