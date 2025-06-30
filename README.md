## Project Overview
This repository delivers a full-featured Amazon Prime clone application, supported by an automated CI/CD pipeline. The deployment runs on AWS EKS, with robust monitoring and management using Argo CD, Prometheus, and Grafana.

- **Terraform**: Automates AWS infrastructure provisioning, such as EC2 instances and EKS clusters.
- **GitHub**: Version control and collaboration platform for managing source code.
- **Jenkins**: Orchestrates CI/CD workflows for seamless automation.
- **SonarQube**: Ensures code quality with static analysis and enforceable quality gates.
- **NPM**: Handles build and dependency management for Node.js projects.
- **Trivy**: Scans containers for security vulnerabilities.
- **Docker**: Packages and deploys application components as portable containers.
- **AWS ECR**: Hosts and manages Docker images securely.
- **AWS EKS**: Manages and scales Kubernetes workloads.
- **ArgoCD**: Automates GitOps-based continuous deployment.
- **Prometheus & Grafana**: Provide comprehensive metrics collection, visualization, and alerting.

## Pre-requisites
1. **AWS Account**: Ensure you have an AWS account. [Create an AWS Account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-creating.html)
2. **AWS CLI**: Install AWS CLI on your local machine. [AWS CLI Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
3. **VS Code (Optional)**: Download and install VS Code as a code editor. [VS Code Download](https://code.visualstudio.com/download)
4. **Install Terraform in Windows**: Download and install Terraform in Windows [Terraform in Windows](https://learn.microsoft.com/en-us/azure/developer/terraform/get-started-windows-bash)

## Configuration
### AWS Setup
1. **IAM User**: Create an IAM user and generate the access and secret keys to configure your machine with AWS.
2. **Key Pair**: Create a key pair named `key` for accessing your EC2 instances.

## Infrastructure Setup Using Terraform
1. **Clone the Repository** (Open Command Prompt & run below):
   ```bash
   git clone https://github.com/AvinashSaxena17/Deploy-amazon-prime-application-on-AWS-EKS.git
   cd Deploy-amazon-prime-application-on-AWS-EKS/
   ```
2. **Initialize and Apply Terraform**:
  - Open `terraform_code/ec2_server/main.tf` in VS Code.
   - Run the following commands:
     ```bash
     aws configure
     terraform init
     terraform apply -auto-approve 
     ```
(for giving auto permission to terraform to create the resources)

This process will provision the EC2 instance and configure the required security groups, while also installing essential tools such as Jenkins, Docker, SonarQube, and more.

## SonarQube Configuration
1. **Login Credentials**: Use `admin` for both username and password.
2. **Generate SonarQube Token**:
   - Create a token under `Administration → Security → Users → Tokens`.
   - Save the token for integration with Jenkins.

## Jenkins Configuration
1. **Add Jenkins Credentials**:
   - Add the SonarQube token, AWS access key, and secret key in `Manage Jenkins → Credentials → System → Global credentials`.
2. **Install Required Plugins**:
   - Install plugins such as SonarQube Scanner, NodeJS, Docker, and Prometheus metrics under `Manage Jenkins → Plugins`.

3. **Global Tool Configuration**:
   - Set up tools like JDK 17, SonarQube Scanner, NodeJS, and Docker under `Manage Jenkins → Global Tool Configuration`.

## Pipeline Overview
### Pipeline Stages
1. **Git Checkout**: Clones the source code from GitHub.
2. **SonarQube Analysis**: Runs static code analysis to detect bugs and code smells.
3. **Quality Gate**: Enforces code quality standards before proceeding.
4. **Install NPM Dependencies**: Installs NodeJS packages.
5. **Trivy Security Scan**: Scans for vulnerabilities in the project.
6. **Docker Build**: Creates a Docker image for the application.
7. **Push to AWS ECR**: Tags and pushes the Docker image to Amazon ECR.
8. **Image Cleanup**: Removes local images on the Jenkins server to free up space.

### Running Jenkins Pipeline
This pipeline will build the project, perform code analysis, and push the resulting Docker image to AWS ECR.

**To set it up, create a Jenkins CI pipeline and add the following script:**

```groovy
pipeline {
    agent any

    tools {
        jdk 'JDK 17'
        nodejs 'NodeJS'
    }

    parameters {
        string(name: 'ECR_REPO_NAME', defaultValue: 'amazon-prime', description: 'Enter your ECR Repository Name')
        string(name: 'AWS_ACCOUNT_ID', defaultValue: '', description: 'Enter your AWS Account ID')
    }

    environment {
        SCANNER_HOME = tool 'SonarQube-scanner'
    }

    stages {
        stage('1. Git checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/AvinashSaxena17/Deploy-amazon-prime-application-on-AWS-EKS.git'
            }
        }

        stage('2. SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=amazon-prime \
                        -Dsonar.projectKey=amazon-prime
                    '''
                }
            }
        }

        stage('3. SonarQube Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false
            }
        }

        stage('4. NPM install') {
            steps {
                sh "npm install"
            }
        }

        stage('5. Trivy Scan') {
            steps {
                sh "trivy fs . > trivy-scan-results.txt"
            }
        }

        stage('6. Docker image build') {
            steps {
                sh "docker build -t ${params.ECR_REPO_NAME}:$BUILD_NUMBER ."
            }
        }

        stage('7. AWS CONFIGURATION') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-acesskey', variable: 'AWS_ACCESS_KEY'),
                    string(credentialsId: 'aws-secretkey', variable: 'AWS_SECRET')
                ]) {
                    sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY
                        aws configure set aws_secret_access_key $AWS_SECRET
                        aws ecr describe-repositories --repository-names ${ECR_REPO_NAME} --region ap-south-1 || \
                        aws ecr create-repository --repository-name ${ECR_REPO_NAME} --region ap-south-1
                    '''
                }
            }
        }

        stage('8. TAG IMAGE') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-acesskey', variable: 'AWS_ACCESS_KEY'),
                    string(credentialsId: 'aws-secretkey', variable: 'AWS_SECRET')
                ]) {
                    sh '''
                        aws ecr get-login-password --region ap-south-1 | \
                        docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com
                        docker tag ${ECR_REPO_NAME}:$BUILD_NUMBER ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/${ECR_REPO_NAME}:$BUILD_NUMBER
                        docker tag ${ECR_REPO_NAME}:$BUILD_NUMBER ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/${ECR_REPO_NAME}:latest
                    '''
                }
            }
        }

        stage('9. PUSH IMAGE') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-acesskey', variable: 'AWS_ACCESS_KEY'),
                    string(credentialsId: 'aws-secretkey', variable: 'AWS_SECRET')
                ]) {
                    sh '''
                        docker push ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/${ECR_REPO_NAME}:$BUILD_NUMBER
                        docker push ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/${ECR_REPO_NAME}:latest
                    '''
                }
            }
        }

        stage('10. CLEANUP IMAGES FROM JENKINS SERVER') {
            steps {
                sh '''
                    docker rmi ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/${ECR_REPO_NAME}:$BUILD_NUMBER
                    docker rmi ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/${ECR_REPO_NAME}:latest
                '''
            }
        }
    }
}

```
## Crete EKS Cluster using terraform :
```
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.15.1"

  cluster_name                   = local.name
  cluster_endpoint_public_access = true

  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
    }
  }

  vpc_id                   = module.vpc.vpc_id
  subnet_ids               = module.vpc.private_subnets

  eks_managed_node_groups = {
    panda-node = {
      min_size     = 2
      max_size     = 4
      desired_size = 2

      instance_types = ["t2.medium"]
      capacity_type  = "SPOT"

      tags = {
        ExtraTag = "Production_Node"
      }
    }
  }

  tags = local.tags
}
```
## Continuous Deployment with ArgoCD
1. **Create EKS Cluster**: Provision an EKS cluster and its associated AWS resources using Terraform.
2. **Deploy Amazon Prime Clone**: Deploy the application to the cluster with ArgoCD, using Kubernetes manifest files.
3. **Monitoring Setup**: Install Prometheus and Grafana via Helm charts to enable monitoring of the Kubernetes cluster.

**Create a Jenkins CD pipeline and add the following script:**
```groovy
pipeline {
    agent any

    environment {
        KUBECTL = '/usr/local/bin/kubectl'
    }

    parameters {
        string(name: 'CLUSTER_NAME', defaultValue: 'amazon-prime-cluster', description: 'Enter your EKS cluster name')
    }

    stages {
        stage("Login to EKS") {
            steps {
                script {
                    withCredentials([string(credentialsId: 'access-key', variable: 'AWS_ACCESS_KEY'),
                                     string(credentialsId: 'secret-key', variable: 'AWS_SECRET_KEY')]) {
                        sh "aws eks --region us-east-1 update-kubeconfig --name ${params.CLUSTER_NAME}"
                    }
                }
            }
        }

        stage("Configure Prometheus & Grafana") {
            steps {
                script {
                    sh """
                    helm repo add stable https://charts.helm.sh/stable || true
                    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                    # Check if namespace 'prometheus' exists
                    if kubectl get namespace prometheus > /dev/null 2>&1; then
                        # If namespace exists, upgrade the Helm release
                        helm upgrade stable prometheus-community/kube-prometheus-stack -n prometheus
                    else
                        # If namespace does not exist, create it and install Helm release
                        kubectl create namespace prometheus
                        helm install stable prometheus-community/kube-prometheus-stack -n prometheus
                    fi
                    kubectl patch svc stable-kube-prometheus-sta-prometheus -n prometheus -p '{"spec": {"type": "LoadBalancer"}}'
                    kubectl patch svc stable-grafana -n prometheus -p '{"spec": {"type": "LoadBalancer"}}'
                    """
                }
            }
        }

        stage("Configure ArgoCD") {
            steps {
                script {
                    sh """
                    # Install ArgoCD
                    kubectl create namespace argocd || true
                    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
                    kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
                    """
                }
            }
        }
		
    }
}
```

## Cleanup
- Run cleanup pipelines to delete the resources such as load balancers, services, and deployment files.

### Cleanup Pipeline
```groovy
pipeline {
    agent any

    environment {
        KUBECTL = '/usr/local/bin/kubectl'
    }

    parameters {
        string(name: 'CLUSTER_NAME', defaultValue: 'amazon-prime-cluster', description: 'Enter your EKS cluster name')
    }

    stages {

        stage("Login to EKS") {
            steps {
                script {
                    withCredentials([string(credentialsId: 'access-key', variable: 'AWS_ACCESS_KEY'),
                                     string(credentialsId: 'secret-key', variable: 'AWS_SECRET_KEY')]) {
                        sh "aws eks --region us-east-1 update-kubeconfig --name ${params.CLUSTER_NAME}"
                    }
                }
            }
        }
        
        stage('Cleanup K8s Resources') {
            steps {
                script {
                    // Step 1: Delete services and deployments
                    sh 'kubectl delete svc kubernetes || true'
                    sh 'kubectl delete deploy pandacloud-app || true'
                    sh 'kubectl delete svc pandacloud-app || true'

                    // Step 2: Delete ArgoCD installation and namespace
                    sh 'kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml || true'
                    sh 'kubectl delete namespace argocd || true'

                    // Step 3: List and uninstall Helm releases in prometheus namespace
                    sh 'helm list -n prometheus || true'
                    sh 'helm uninstall kube-stack -n prometheus || true'
                    
                    // Step 4: Delete prometheus namespace
                    sh 'kubectl delete namespace prometheus || true'

                    // Step 5: Remove Helm repositories
                    sh 'helm repo remove stable || true'
                    sh 'helm repo remove prometheus-community || true'
                }
            }
        }
		
        stage('Delete ECR Repository and KMS Keys') {
            steps {
                script {
                    // Step 1: Delete ECR Repository
                    sh '''
                    aws ecr delete-repository --repository-name amazon-prime --region us-east-1 --force
                    '''

                    // Step 2: Delete KMS Keys
                    sh '''
                    for key in $(aws kms list-keys --region us-east-1 --query "Keys[*].KeyId" --output text); do
                        aws kms disable-key --key-id $key --region us-east-1
                        aws kms schedule-key-deletion --key-id $key --pending-window-in-days 7 --region us-east-1
                    done
                    '''
                }
            }
        }		
		
    }
}
```
**This action uses the terraform destroy command to immediately destroy all resources in a Terraform workspace.**
```
terraform destroy -auto-approve
```


