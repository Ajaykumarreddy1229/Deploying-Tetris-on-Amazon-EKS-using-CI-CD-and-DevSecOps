# Project Documentation
## Deploying Tetris on Amazon EKS using CI/CD and DevSecOps with Jenkins, Terraform and Argo CD

## 1. Project Overview

The purpose of this project is to deploy the Tetris application on Amazon EKS using a CI/CD and GitOps workflow.

The project uses three GitHub repositories:

1. **Version 1 application repository** – contains the first version of the Tetris application.
2. **Version 2 application repository** – contains the updated Tetris application.
3. **Kubernetes manifest repository** – contains `deployment.yml`, which stores the Docker image reference used by Kubernetes and is connected to Argo CD.

For each application version, Jenkins checks out the code, performs code-quality and security checks, builds a Docker image, and pushes the image to Docker Hub. Jenkins then updates the image reference in the manifest repository. Argo CD monitors that repository and automatically synchronizes the change to the EKS cluster.

## 2. GitHub Repositories

| Repository | Role |
|---|---|
| https://github.com/Ajaykumarreddy1229/devsecops-Tetris-V1 | Tetris Version 1 source |
| https://github.com/Ajaykumarreddy1229/devsecops-tetris-v2 | Tetris Version 2 source |
| https://github.com/Ajaykumarreddy1229/devsecops-Tetris-manifest | Kubernetes manifest repository |

## 3. Architecture and Workflow

The architecture image is stored as `architecture.gif` beside this document.

![Tetris CI/CD architecture](architecture.gif)

### Workflow

1. Application source code is stored in GitHub.
2. Jenkins checks out the selected application version.
3. SonarQube performs static code analysis.
4. Jenkins waits for the SonarQube Quality Gate result.
5. Jenkins installs application dependencies with `npm install`.
6. Trivy scans the source filesystem.
7. Docker builds the application image.
8. Jenkins pushes the image to Docker Hub.
9. Trivy scans the pushed Docker image.
10. Jenkins checks out the Kubernetes manifest repository.
11. Jenkins updates the image in `deployment.yml` and pushes the commit to GitHub.
12. Argo CD detects the Git change and synchronizes the Kubernetes resources on EKS.
13. The Tetris application is exposed through a Kubernetes Service of type `LoadBalancer`.

## 4. AWS EC2 Setup

The notes specify the following EC2 setup:

- Instance type: `m7i-flex.large`
- Operating system: Ubuntu 24
- Storage: 50 GB
- IAM role: an IAM role was attached to the instance for AWS access.

The EC2 instance is used to run Jenkins and the supporting DevSecOps tools.

**IAM note:** The original setup notes mention an Administrator-access role. For a safer implementation, use a dedicated IAM role with only the permissions required for the Terraform and EKS operations. Do not use permanent access keys in scripts or source control.

## 5. Installing Required Tools

The setup uses two shell scripts.

### 5.1 Script 1 – Java, Jenkins and Docker

The first script performs these tasks:

- Updates Ubuntu packages.
- Installs OpenJDK 21, `wget`, and `curl`.
- Downloads Jenkins WAR version `2.552`.
- Starts Jenkins using Java in the background.
- Writes Jenkins output to `/opt/jenkins/jenkins.log`.
- Installs Docker Engine and its related plugins.
- Adds the `ubuntu` user to the Docker group.

The Jenkins URL used in the notes is:

`http://<SERVER-IP>:8080`

The Jenkins WAR is started with `nohup`. This is the method used in the supplied setup notes; it is not a full system-service configuration. A production setup should use a managed service, persistent logs, and appropriate access restrictions.

### 5.2 Script 2 – Terraform, kubectl, AWS CLI and Trivy

The second script installs:

- Terraform from the HashiCorp APT repository.
- `kubectl` for Kubernetes cluster management.
- AWS CLI v2.
- Trivy for filesystem and container-image scanning.

Verify each installation after running the script:

```bash
terraform version
kubectl version --client
aws --version
trivy --version
```

The notes use the command `which terraform` to find the Terraform executable path for Jenkins tool configuration.

## 6. SonarQube Setup

SonarQube is run as a Docker container:

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Open the SonarQube interface at:

`http://<SERVER-IP>:9000`

The supplied notes use the initial login `admin` / `admin`. Change the initial password during setup and do not publish credentials.

### Configure the SonarQube token

1. Open SonarQube Administration.
2. Go to Security and Users.
3. Generate a token for the project.
4. Copy the token securely.
5. In Jenkins, open **Manage Jenkins → Credentials**.
6. Add a **Secret text** credential.
7. Use the credential ID `sonar-token`.

### Configure the SonarQube server in Jenkins

Under **Manage Jenkins → System → SonarQube servers**, configure:

- Name: `sonar-server`
- Server URL: the SonarQube server URL
- Server authentication token: `sonar-token`

### Configure the Jenkins webhook

In SonarQube, open **Administration → Configuration → Webhooks** and configure the Jenkins webhook URL:

`http://<JENKINS-SERVER>:8080/sonarqube-webhook/`

The supplied notes leave the webhook secret empty. If a secret is configured, configure the matching secret in Jenkins as well.

## 7. Jenkins Configuration

### 7.1 Terraform tool

1. Run `which terraform` on the Jenkins EC2 instance.
2. Open **Manage Jenkins → Tools**.
3. Add a Terraform installation named `terraform`.
4. Disable automatic installation if using the system-installed executable.
5. Set the installation path according to the actual Terraform executable location.

### 7.2 Plugins listed in the setup notes

The notes list these Jenkins plugins:

- Eclipse Temurin installer
- SonarQube Scanner
- NodeJS
- OWASP Dependency-Check
- Docker
- Docker Commons
- Docker Pipeline
- Docker API
- Docker-build-step

Install the plugins from **Manage Jenkins → Plugins** and configure the corresponding tools under **Manage Jenkins → Tools**.

The supplied configuration notes refer to tool names such as `jdk25`, `node25`, `sonar-scanner`, and `DP-Check`. They also mention selecting JDK 24 from Adoptium while naming the Jenkins tool `jdk25`. Ensure the configured tool name and actual installed version are consistent with the Jenkinsfile.

### 7.3 Docker Hub credentials

In **Manage Jenkins → Credentials**, add a username/password credential:

- Username: your Docker Hub username
- Password: Docker Hub access token or password
- ID: `docker`

Use the credential ID in the Jenkins pipeline. Never place the password directly in a Jenkinsfile.

### 7.4 GitHub credentials

Create a GitHub Personal Access Token with only the permissions required to update the manifest repository. Store it in Jenkins as a **Secret text** credential with ID:

`github`

The project notes mention classic-token permissions including `repo`, `write:packages`, and `admin:repo`. Do not automatically grant broad permissions if they are not required. Use the narrowest supported permissions for the repository operation.

**Important:** A token was present in the original working notes. It is intentionally not reproduced here. If that token was real, revoke it and create a replacement.

## 8. Jenkins Job 1 – EKS Provisioning

Create a Jenkins Pipeline job named:

`EKSJOB`

The notes specify a parameterized job with:

- Parameter name: `action`
- Choices: `apply`, `destroy`

The pipeline checks out the Version 1 repository and runs Terraform from the `EKS_TERRAFORM` directory.

### Pipeline stages

1. **Checkout from Git** – checks out the Terraform project.
2. **Terraform version** – checks the installed Terraform version.
3. **Terraform init** – initializes the Terraform working directory.
4. **Terraform validate** – validates the Terraform configuration.
5. **Terraform plan** – previews the infrastructure changes.
6. **Terraform apply/destroy** – runs the selected action.

Before running the job, update the S3 bucket name in:

`EKS_TERRAFORM/backend.tf`

The bucket must be one you own and have configured for the Terraform backend. The supplied notes do not include the Terraform resource files or backend configuration, so those files must be maintained in the repository.

**Operational caution:** `destroy` can remove infrastructure. Use a separate approval step and verify the selected workspace and account before running it.

## 9. Jenkins Job 2 – Tetris Version 1

Create a pipeline job named:

`TETRISV1`

The supplied pipeline uses these Jenkins tools:

- JDK tool: `jdk25`
- NodeJS tool: `node25`
- SonarQube Scanner: `sonar-scanner`

### Pipeline stages

#### Stage 1 – Clean workspace

Removes files from the previous build workspace using `cleanWs()`.

#### Stage 2 – Checkout from Git

Checks out the `main` branch of:

`https://github.com/Ajaykumarreddy1229/devsecops-Tetris-V1.git`

#### Stage 3 – SonarQube analysis

Runs the SonarQube scanner with the project name and key:

- Project name: `TetrisVersion1.0`
- Project key: `TetrisVersion1.0`

#### Stage 4 – Quality Gate

Uses `waitForQualityGate` to receive the Quality Gate result from SonarQube.

The supplied pipeline uses `abortPipeline: false`, which means the pipeline is not configured to stop automatically when the gate fails. Change this behavior if the required release policy is to block deployments on a failed Quality Gate.

#### Stage 5 – Install dependencies

Runs:

```bash
npm install
```

#### Stage 6 – Trivy filesystem scan

The later pipeline version includes:

```bash
trivy fs . > trivyfs.txt
```

This writes the filesystem scan output to `trivyfs.txt` in the Jenkins workspace.

#### Stage 7 – Docker build and push

Builds the application image and pushes it to Docker Hub. The Version 1 image reference used in the notes is:

`ajay1229/tetrisv1:latest`

#### Stage 8 – Trivy image scan

Runs:

```bash
trivy image ajay1229/tetrisv1:latest > trivyimage.txt
```

The scan output is written to `trivyimage.txt`.

#### Stage 9 – Checkout the manifest repository

Checks out:

`https://github.com/Ajaykumarreddy1229/devsecops-Tetris-manifest.git`

#### Stage 10 – Update deployment image

Jenkins updates the image line in `deployment.yml`, commits the change, and pushes it to the manifest repository using the `github` Jenkins credential.

The intended Version 1 image reference is:

`ajay1229/tetrisv1:latest`

**Check the supplied pipeline before running:** one earlier image-updater example contains a different Docker Hub namespace (`mukulranjan27`) in the image variable and tag command, while the push command uses `ajay1229`. Keep the build, push, and manifest image references consistent. The manifest must point to the image that was actually pushed.

## 10. Configure EKS Access from Jenkins

After the EKS cluster is created, configure the kubeconfig on the Jenkins instance:

```bash
aws eks update-kubeconfig --name EKS_CLUSTER --region ap-south-1
```

Check the cluster nodes:

```bash
kubectl get nodes
```

The notes also use:

```bash
cat /root/.kube/config
```

Do not copy kubeconfig contents into GitHub or expose them in build logs. Give the Jenkins identity only the Kubernetes permissions it needs.

## 11. Argo CD Installation and Configuration

### 11.1 Create the Argo CD namespace

```bash
kubectl create namespace argocd
```

### 11.2 Install Argo CD

The supplied notes use Argo CD manifest version `v2.4.7`:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
```

This is the version in the original project notes. Review the supported version and upgrade guidance before using it for a new production deployment.

### 11.3 Expose the Argo CD server

The notes expose the server through an AWS LoadBalancer:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Check the service:

```bash
kubectl get svc -n argocd
```

This creates an externally reachable endpoint. Restrict access using appropriate network controls and authentication; do not leave the administrative interface open to everyone.

### 11.4 Retrieve the initial admin password

The notes use:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Use the resulting password to sign in as `admin`, then change it and store it securely. Do not put it in GitHub or pipeline logs.

### 11.5 Connect the GitHub manifest repository

In the Argo CD interface:

1. Open **Settings → Repositories**.
2. Select **Connect Repo Using HTTPS**.
3. Select repository type `git`.
4. Select project `default`.
5. Enter the manifest repository URL:

   `https://github.com/Ajaykumarreddy1229/devsecops-Tetris-manifest.git`

6. Provide authentication if the repository requires it.

### 11.6 Create the Tetris application

In Argo CD, create an application with the settings from the notes:

| Setting | Value |
|---|---|
| Application name | `tetris` |
| Project | `default` |
| Sync policy | `Automatic` |
| Repository | Tetris manifest repository |
| Revision | `HEAD` |
| Path | `./` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

After creating the application, Argo CD reads the Kubernetes manifests from the repository and synchronizes the resources to the EKS cluster.

## 12. Access the Tetris Application

The Kubernetes manifest defines the application resources. The notes describe a Service of type `LoadBalancer`, which provides an AWS load balancer endpoint.

Check the Kubernetes resources:

```bash
kubectl get all
```

Open the external hostname shown for the Tetris Service in a browser to access the game.

The hostname in the original notes is an example from that deployment. A new deployment will normally receive its own load balancer hostname.

## 13. Deploying Tetris Version 2

Version 2 uses a separate application repository:

`https://github.com/Ajaykumarreddy1229/devsecops-tetris-v2.git`

Create a separate Jenkins pipeline for Version 2. The stages follow the Version 1 pipeline:

1. Clean the Jenkins workspace.
2. Check out the Version 2 source repository.
3. Run SonarQube analysis using project name/key `TetrisVersion2.0`.
4. Wait for the Quality Gate result.
5. Install dependencies using `npm install`.
6. Run the Trivy filesystem scan.
7. Build the Version 2 Docker image.
8. Push the image to Docker Hub.
9. Scan the Docker image with Trivy.
10. Check out the Kubernetes manifest repository.
11. Update `deployment.yml` to use the Version 2 image.
12. Commit and push the manifest change to GitHub.

The Version 2 image reference in the supplied notes is:

`ajay1229/tetrisv2:latest`

When Jenkins pushes the updated manifest, Argo CD detects the Git change and automatically synchronizes the updated deployment to EKS. Refresh the application endpoint to check the new version.

## 14. Security and Configuration Notes

These points are important when reproducing the project:

- **Credentials:** The original notes contain a GitHub token. It is not included in this document. Revoke it if it was real. Store replacement tokens and all other secrets in Jenkins Credentials.
- **IAM:** Use a dedicated least-privilege IAM role rather than broad administrator access.
- **Docker socket:** The original notes include `chmod 777 /var/run/docker.sock`. This grants very broad access to the Docker daemon. Avoid this setting; configure restricted group or agent access instead.
- **Jenkins exposure:** Restrict port `8080` to trusted IPs or a private network and use HTTPS for shared environments.
- **SonarQube and Argo CD:** Restrict their web interfaces and rotate initial credentials.
- **Trivy findings:** Save scan reports as build artifacts and review vulnerabilities. A scan command that only writes a report does not automatically fail the pipeline on findings; configure an appropriate severity threshold and exit-code policy if builds must be blocked.
- **OWASP Dependency-Check:** The plugin is listed in the setup notes, but the Jenkins pipeline excerpts supplied here do not show a Dependency-Check execution stage. Add and configure that stage if you want the pipeline to run it.
- **Docker Scout:** It appears in the initial tool/stage description, but the detailed pipeline commands shown use Trivy. This document describes the implemented scan commands as Trivy and does not claim Docker Scout was executed.
- **Tool versions:** Confirm the JDK and NodeJS versions configured in Jenkins match the tool names referenced by the Jenkinsfiles.
- **Image naming:** Use the same Docker Hub namespace and image tag for build, push, scan, and Kubernetes deployment.
- **Terraform destroy:** Protect destructive actions with manual approval and account/environment checks.

## 15. Project Result

The project demonstrates an automated application delivery flow:

- Terraform provisions the EKS infrastructure through Jenkins.
- Jenkins performs application checkout, code-quality analysis, dependency installation, container build, and vulnerability scanning.
- Docker images are pushed to Docker Hub.
- Jenkins updates the Kubernetes manifest repository.
- Argo CD monitors Git and automatically synchronizes the desired application state to EKS.
- The same workflow is used to move from Tetris Version 1 to Version 2.

## 16. Learning Outcomes

By completing this project, I practised:

- AWS EC2 and IAM role configuration.
- Terraform-based EKS provisioning.
- Jenkins installation and pipeline creation.
- GitHub integration and credential management.
- SonarQube static code analysis and Quality Gate integration.
- Docker image build and Docker Hub push.
- Trivy filesystem and image vulnerability scanning.
- Kubernetes access using `kubectl`.
- Argo CD installation, repository connection, and automatic synchronization.
- GitOps-based application updates from Version 1 to Version 2.
