# DevOps Final Project: Automated CI/CD Deployment on Amazon EKS

A DevOps project demonstrating automated application delivery using
**GitHub, Jenkins, Docker, Amazon ECR, Amazon EKS, Kubernetes, and the
AWS Load Balancer Controller**.

> This README documents the project architecture, prerequisites, setup,
> pipeline workflow, validation, and troubleshooting. Application code,
> Docker configuration, Jenkins pipeline, and Kubernetes manifests are
> maintained as separate files in this repository; their contents are
> not duplicated here.

## Table of Contents

-   [Project Overview](#project-overview)
-   [Architecture and Workflow](#architecture-and-workflow)
-   [Technology Stack](#technology-stack)
-   [Repository Structure](#repository-structure)
-   [Prerequisites](#prerequisites)
-   [AWS Configuration](#aws-configuration)
-   [Jenkins Configuration](#jenkins-configuration)
-   [Configure the Kubernetes
    Ingress](#configure-the-kubernetes-ingress)
-   [Create the Jenkins Pipeline Job](#create-the-jenkins-pipeline-job)
-   [Validate the Deployment](#validate-the-deployment)
-   [Troubleshooting](#troubleshooting)
-   [Security and Cost Notes](#security-and-cost-notes)
-   [Cleanup](#cleanup)
-   [Project Outcome](#project-outcome)

## Project Overview

The goal is to automate the deployment of a frontend application to
Amazon EKS. When a developer pushes changes to GitHub, Jenkins runs the
pipeline, validates the required project files, builds a Docker image,
pushes that image to Amazon ECR, and updates the Kubernetes application
running on EKS.

The application is exposed externally through a Kubernetes Ingress
managed by the AWS Load Balancer Controller, which provisions an AWS
Application Load Balancer (ALB).

## Architecture and Workflow

``` text
Developer
   |
   | Push code
   v
GitHub Repository
   |
   | Jenkins pipeline
   v
Jenkins on Ubuntu EC2
   |
   |-- Validate project files
   |-- Build Docker image
   |-- Authenticate to Amazon ECR
   |-- Push versioned image
   |-- Update Kubernetes deployment
   v
Amazon ECR
   |
   | Image pulled by worker nodes
   v
Amazon EKS
   |
   | Deployment -> Pods -> ClusterIP Service
   |                         |
   |                      Ingress
   v
AWS Load Balancer Controller
   |
   v
Application Load Balancer (ALB)
   |
   v
User's browser
```

### CI/CD workflow

1.  The developer commits and pushes code to GitHub.
2.  Jenkins checks out the repository.
3.  Jenkins validates that the required application files exist.
4.  Docker builds an image tagged with the Jenkins build number.
5.  Jenkins authenticates to Amazon ECR and pushes the image.
6.  Jenkins updates the Kubernetes Deployment to use the new image.
7.  Kubernetes performs a rolling update of the application Pods.
8.  Jenkins checks the rollout status and displays deployment
    information.
9.  The ALB routes HTTP traffic to the application through the
    Kubernetes Service.

## Technology Stack

  -----------------------------------------------------------------------
  Technology                          Purpose
  ----------------------------------- -----------------------------------
  Git and GitHub                      Version control and source code
                                      hosting

  Jenkins                             Continuous integration and
                                      continuous deployment

  Docker                              Packages the application into a
                                      container image

  Amazon ECR                          Stores versioned container images

  Amazon EKS                          Managed Kubernetes platform

  Kubernetes                          Runs and manages application
                                      workloads

  AWS Load Balancer Controller        Connects Kubernetes Ingress
                                      resources to AWS load balancers

  Application Load Balancer           Routes external HTTP traffic to the
                                      application

  Prometheus and Grafana (optional)   Monitoring and visualization
                                      extension
  -----------------------------------------------------------------------

**Scope:** This project does not require Terraform, Ansible, or Trivy.

## Repository Structure

The repository contains the application and deployment files. Their
contents are maintained separately and are not reproduced in this
README.

``` text
devops-final-project/
├── index.html
├── style.css
├── script.js
├── Dockerfile
├── Jenkinsfile
├── .dockerignore
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

## Prerequisites

Before deployment, ensure you have:

-   A GitHub repository containing the project files.
-   An active AWS account with access to EC2, ECR, EKS, IAM, and Elastic
    Load Balancing.
-   An Ubuntu EC2 instance for Jenkins.
-   Jenkins with the Pipeline and Git plugins.
-   Docker Engine, AWS CLI v2, and `kubectl` installed on the Jenkins
    agent.
-   An ECR repository for the application image.
-   A working EKS cluster with healthy worker nodes.
-   Appropriate IAM permissions and EKS cluster access for Jenkins.
-   The AWS Load Balancer Controller installed and configured in the
    cluster.
-   VPC subnets and routing configured for the intended ALB scheme.

## Project Configuration

The examples in this repository use the following values. Change them if
your actual AWS resources use different names.

  Setting                 Example value
  ----------------------- ---------------------------
  AWS Region              `ap-south-1`
  ECR Repository          `devops-final-project`
  EKS Cluster             `devops-final-cluster`
  Kubernetes Deployment   `devops-frontend`
  Kubernetes container    `frontend`
  Kubernetes Service      `devops-frontend-service`
  Kubernetes Ingress      `devops-frontend-ingress`
  Kubernetes namespace    `default`

The ECR registry URI follows this format:

`<AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/<ECR_REPOSITORY>`

Use your actual AWS account ID rather than assuming the example account
ID is correct.

## AWS Configuration

### 1. Verify AWS access

Use AWS CLI on the machine where you administer AWS resources:

``` bash
aws --version
aws sts get-caller-identity
aws configure list
```

Confirm that the returned identity belongs to the intended AWS account
and has the required permissions.

### 2. Create or verify the ECR repository

Check whether the repository already exists:

``` bash
aws ecr describe-repositories \
  --repository-names devops-final-project \
  --region ap-south-1
```

If it does not exist, create it:

``` bash
aws ecr create-repository \
  --repository-name devops-final-project \
  --region ap-south-1
```

### 3. Verify the EKS cluster

``` bash
aws eks list-clusters --region ap-south-1
aws eks describe-cluster \
  --name devops-final-cluster \
  --region ap-south-1
```

Configure `kubectl` to access the intended cluster:

``` bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name devops-final-cluster

kubectl get nodes
```

Proceed only when the cluster exists and its worker nodes are healthy.
Do not create another cluster if the intended cluster already exists.

### 4. Configure Jenkins permissions

Prefer an IAM role attached to the Jenkins EC2 instance over long-lived
AWS access keys. The Jenkins identity needs appropriate permissions to
authenticate and push images to the intended ECR repository, describe
the EKS cluster, and access the Kubernetes API.

AWS IAM permissions do not automatically grant Kubernetes permissions.
Configure EKS access for the Jenkins IAM principal and grant only the
Kubernetes permissions needed to manage the application's Deployment,
Service, and Ingress.

The EKS worker-node role must also have permissions to pull the image
from ECR.

## Jenkins Configuration

### 1. Install and secure Jenkins

Install a supported Jenkins LTS release on Ubuntu by following the
official instructions:

-   [Jenkins installation
    guide](https://www.jenkins.io/doc/book/installing/linux/)
-   [Jenkins plugin
    documentation](https://www.jenkins.io/doc/book/managing/plugins/)

Restrict SSH and Jenkins UI access to trusted IP addresses. Avoid
exposing an unsecured Jenkins instance to the public internet.

### 2. Install required tools

The Jenkins build agent needs Docker, AWS CLI v2, and `kubectl`. Verify
these tools using the same operating-system account that runs the
Jenkins job.

Docker access from Jenkins must be configured carefully. Access to the
Docker socket can provide extensive privileges on the host, so use a
dedicated, trusted build machine.

### 3. Check the Jenkinsfile

The `Jenkinsfile` must be in the repository root, and the Jenkins job
must reference that exact path.

**Important:** The actual Jenkinsfile must not contain Markdown code
fences. Its first line should be the pipeline declaration, not a line
such as ```` ```groovy ````.

The pipeline should use a unique image tag for each build, push the
image before updating Kubernetes, and verify the Deployment rollout
after applying the manifests.

## Configure the Kubernetes Ingress

The Kubernetes Ingress resource does not create an ALB by itself. The
AWS Load Balancer Controller must be installed and configured for the
EKS cluster.

Follow the current official documentation:

-   [AWS Load Balancer Controller on
    EKS](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)
-   [AWS Load Balancer Controller installation
    guide](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)

Check that the controller is deployed and running:

``` bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get pods -n kube-system
```

The controller requires the correct IAM permissions. The VPC subnets
must also have appropriate tags and routing. An internet-facing ALB
requires suitable public subnets; use the correct scheme and networking
for private deployments.

## Create the Jenkins Pipeline Job

1.  Push the project files to the GitHub repository.

2.  Open Jenkins and select **New Item**.

3.  Enter a job name such as `devops-final-project`.

4.  Choose **Pipeline** and select **OK**.

5.  In the Pipeline section, set **Definition** to **Pipeline script
    from SCM**.

6.  Select **Git** and enter the repository URL:

    `https://github.com/vekhandesushant-pngsushant/devops-final-project.git`

7.  Set the branch to `*/main`.

8.  Set the script path to `Jenkinsfile`.

9.  Save the job and select **Build Now**.

10. Review **Console Output** and confirm each pipeline stage completes
    successfully.

To trigger builds automatically after pushes, configure a GitHub webhook
and the matching Jenkins trigger/plugin. Ensure the Jenkins endpoint is
securely reachable before enabling inbound webhook traffic. SCM polling
is another option where appropriate.

## Validate the Deployment

After a successful pipeline run, use `kubectl` to verify the
application:

``` bash
kubectl get nodes
kubectl get pods
kubectl get deployment devops-frontend
kubectl get service devops-frontend-service
kubectl get ingress devops-frontend-ingress
kubectl rollout status deployment/devops-frontend
```

Wait for the Ingress to receive an ALB hostname:

``` bash
kubectl get ingress devops-frontend-ingress --watch
```

When the Ingress `ADDRESS` field contains the ALB hostname, open it in a
browser. ALB provisioning may take several minutes.

## Troubleshooting

  ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  Problem                                                                                                                                                                                                                                                                                                                                                                                                                        What to check
  ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -----------------------------------
  Jenkins reports `unexpected char: '`'`| Remove Markdown code-fence lines from the actual Jenkinsfile. | | Docker permission denied | Confirm Docker is running and Jenkins has the required Docker access. | |`dpkg`lock is held | Wait for the active package update. Never delete package-manager lock files. | | AWS credentials unavailable | Verify the Jenkins IAM role and run`aws sts                                  
  get-caller-identity`as the Jenkins service user. | | ECR login or push fails | Check the AWS account ID, region, repository name, and ECR permissions. | | EKS cluster not found | Confirm the cluster name and region; check`aws eks                                                                                                                                                                                          
  list-clusters`. | |`kubectl`unauthorized/forbidden | Configure EKS access for the Jenkins IAM principal and grant required Kubernetes permissions. | | Pods show`ImagePullBackOff`| Verify the image URI and tag, and confirm worker-node ECR pull permissions. | | Ingress has no ALB address | Check controller health/logs, IAM permissions, subnet tags, and VPC routing. | | Deployment rollout fails | Inspect`kubectl   
  describe deployment\`, Pod events, and application logs.                                                                                                                                                                                                                                                                                                                                                                       

  ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Useful diagnostic commands:

``` bash
kubectl describe deployment devops-frontend
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs deployment/devops-frontend
kubectl describe ingress devops-frontend-ingress
kubectl logs deployment/aws-load-balancer-controller -n kube-system
```

Replace `<pod-name>` with the actual Pod name.

## Optional Monitoring Extension

Prometheus and Grafana can be added to monitor Kubernetes and
application infrastructure. This is optional and not required for the
core CI/CD workflow.

The `kube-prometheus-stack` Helm chart is documented here:

-   [Prometheus Community Helm
    charts](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

Review resource requirements and chart values before installation. Avoid
exposing Grafana publicly without appropriate authentication and TLS.

## Security and Cost Notes

-   Do not commit AWS access keys, secret keys, passwords, or private
    keys to GitHub.
-   Prefer IAM roles and least-privilege permissions.
-   Restrict access to Jenkins and secure its credentials.
-   Use public ALBs only when public access is intended.
-   AWS resources such as EKS, EC2, ECR storage, ALBs, NAT gateways, and
    monitoring infrastructure can incur charges. Review current AWS
    pricing before creating resources.
-   Confirm that AWS account activation and service access are complete
    before deployment.

## Cleanup

Delete only resources created for this project, and verify that you no
longer need them.

Remove the application's Kubernetes resources:

``` bash
kubectl delete -f k8s/ingress.yaml
kubectl delete -f k8s/service.yaml
kubectl delete -f k8s/deployment.yaml
```

If the EKS cluster was created only for this project and is no longer
needed, delete it using the tool with which it was created. For an
`eksctl`-managed cluster:

``` bash
eksctl delete cluster \
  --name devops-final-cluster \
  --region ap-south-1
```

If you no longer need the ECR repository or its images:

``` bash
aws ecr delete-repository \
  --repository-name devops-final-project \
  --region ap-south-1 \
  --force
```

Also review the AWS Console for remaining EC2 instances, EBS volumes,
load balancers, Elastic IPs, NAT gateways, and monitoring resources
created for the lab. Deleting the EKS cluster does not necessarily
delete every separately created resource.

## Final Checklist

-   [ ] Application builds and runs locally in Docker.
-   [ ] Project files are pushed to GitHub.
-   [ ] Jenkins can access the repository and execute the pipeline.
-   [ ] ECR repository exists and receives versioned images.
-   [ ] EKS cluster and worker nodes are healthy.
-   [ ] Jenkins has appropriate ECR and EKS/Kubernetes access.
-   [ ] AWS Load Balancer Controller is healthy.
-   [ ] Kubernetes Deployment rollout succeeds.
-   [ ] Ingress receives an ALB hostname.
-   [ ] Application opens through the ALB.
-   [ ] Unneeded AWS resources are cleaned up.

## Project Outcome

This project demonstrates a practical DevOps delivery workflow: source
control with GitHub, automated CI/CD with Jenkins, containerization with
Docker, image storage in Amazon ECR, Kubernetes deployment on Amazon
EKS, and external traffic routing through an AWS Application Load
Balancer.

For a portfolio or final evaluation, consider adding screenshots of the
successful Jenkins pipeline, ECR image, Kubernetes Pods and Service,
Ingress/ALB hostname, and the application in a browser. Never include
credentials or other sensitive account information in public
screenshots.
