# Deploying-Tetris-on-Amazon-EKS-using-CI-CD-and-DevSecOps


## Project Overview

This project demonstrates how to build and deploy the Tetris application on **Amazon EKS** using a Jenkins CI/CD pipeline and Argo CD GitOps.

The pipeline checks the application code, performs code-quality and security scans, builds a Docker image, and pushes the image to Docker Hub. Jenkins then updates the Kubernetes deployment manifest in a separate GitHub repository. Argo CD monitors that repository and automatically synchronizes the updated manifest with the EKS cluster.

The project demonstrates two application versions:

- **Tetris Version 1:** Builds and pushes the first Docker image and updates the Kubernetes manifest.
- **Tetris Version 2:** Builds and pushes the second Docker image and updates the same manifest repository. Argo CD detects the change and deploys the updated version.

## Architecture

<img width="1038" height="545" alt="Architecture" src="https://github.com/user-attachments/assets/7c9cf190-c217-4ab8-b29b-a0ca53570d41" />


## Project Repositories

| Repository | Purpose |
|---|---|
| [Tetris Version 1](https://github.com/Ajaykumarreddy1229/devsecops-Tetris-V1) | Version 1 application source and pipeline |
| [Tetris Version 2](https://github.com/Ajaykumarreddy1229/devsecops-tetris-v2) | Version 2 application source and pipeline |
| [Kubernetes Manifest](https://github.com/Ajaykumarreddy1229/devsecops-Tetris-manifest) | Kubernetes deployment manifest monitored by Argo CD |

## Tools and Technologies

- **AWS EC2:** Hosts the Jenkins and supporting tools.
- **Amazon EKS:** Runs the Kubernetes application.
- **Terraform:** Provisions the EKS cluster and node group.
- **Jenkins:** Runs the CI/CD pipelines.
- **GitHub:** Stores application code and Kubernetes manifests.
- **SonarQube:** Performs static code-quality analysis.
- **OWASP Dependency-Check:** Plugin installed in Jenkins for dependency checking.
- **Trivy:** Scans the source filesystem and Docker images for vulnerabilities.
- **Docker:** Builds the application container image.
- **Docker Hub:** Stores the versioned application images.
- **kubectl:** Connects to and manages the EKS cluster.
- **Argo CD:** Watches the manifest repository and synchronizes Kubernetes resources.

## CI/CD Workflow

1. Developer code is stored in the Version 1 or Version 2 GitHub repository.
2. Jenkins checks out the application source code.
3. Jenkins runs SonarQube analysis and checks the Quality Gate.
4. Jenkins installs application dependencies using `npm install`.
5. Trivy scans the source filesystem.
6. Jenkins builds the Docker image and pushes it to Docker Hub.
7. Trivy scans the Docker image.
8. Jenkins checks out the Kubernetes manifest repository.
9. Jenkins updates the image reference in `deployment.yml` and pushes the change to GitHub.
10. Argo CD detects the manifest change and automatically synchronizes the application with Amazon EKS.
11. The application is accessed through the Kubernetes Service LoadBalancer.

## Version Update Workflow

The Version 2 pipeline follows the same process as Version 1, but uses the Version 2 source repository and image name.

When Jenkins updates the image reference in the manifest repository, Argo CD detects the Git change and deploys the new image to EKS. This demonstrates GitOps-based application updates without manually applying the deployment manifest for each release.

## Main Project Activities

- Create an EC2 instance for Jenkins and supporting tools.
- Install Java, Jenkins, Docker, Terraform, AWS CLI, kubectl, and Trivy.
- Run SonarQube as a Docker container.
- Provision the EKS cluster using Terraform from Jenkins.
- Configure Jenkins credentials and tool installations.
- Create Jenkins pipelines for EKS provisioning and application deployment.
- Build and push Tetris Docker images to Docker Hub.
- Update Kubernetes manifests through Jenkins.
- Install and configure Argo CD on EKS.
- Connect Argo CD to the Kubernetes manifest repository.
- Enable automatic synchronization.
- Deploy Version 1 and update the application to Version 2.

## Detailed Documentation

Seefor the setup process, pipeline stages, configuration steps, deployment workflow, and important security notes.

## Important Security Notes

- Do not commit GitHub tokens, Docker Hub passwords, SonarQube tokens, kubeconfig files, or other credentials to GitHub.
- Store credentials in Jenkins Credentials and use credential IDs in pipeline code.
- Use a least-privilege IAM role instead of permanent administrator-level access.
- Restrict access to Jenkins, SonarQube, and Argo CD.
- Avoid giving every user unrestricted access to `/var/run/docker.sock`.
- Review Trivy findings before promoting an image to deployment.

## Learning Outcomes

Through this project, I practised infrastructure provisioning with Terraform, CI/CD automation with Jenkins, code-quality analysis with SonarQube, container image creation with Docker, vulnerability scanning with Trivy, Kubernetes deployment on Amazon EKS, and GitOps delivery with Argo CD.
