
Blue-Green Deployment Documentation
Objective
The best thing about Blue-Green Deployment is that it allows zero downtime when upgrading application versions. This ensures a smooth user experience and makes rollbacks quick and easy if needed.
________________________________________
Architecture Diagram
The application setup includes:
•	Application Components: 
o	MySQL Database (ClusterIP Service)
o	Java-based Application Pods (V1 and V2)
o	Service Load Balancer (Handling traffic routing between Blue and Green environments)
•	How It Works: 
o	The application pods send requests to the MySQL ClusterIP Service.
o	The Load Balancer directs user traffic to either the Blue (V1) or Green (V2) environment.
________________________________________
Step-by-Step Implementation
1. Set Up EC2 Instances
•	Created 4 EC2 instances: 
o	Server (for Terraform & Kubernetes)
o	Jenkins
o	SonarQube
o	Nexus
•	Opened necessary ports: 
o	TCP: 1000-11000, 500-1000
o	HTTP, HTTPS, SSH
2. Connect to Instances
•	Used MobaXterm and .pem key to connect.
•	Updated system: 
•	sudo apt update
•	Configured AWS CLI: 
•	aws configure
3. Install Terraform & Create EKS Cluster
•	Installed Terraform on the Server instance.
•	Cloned the GitHub repository: 
•	git clone <repo-url>
•	cd Bleu-Green-Deployment/Cluster
•	Initialized and deployed EKS cluster: 
•	terraform init
•	terraform plan
•	terraform apply -auto-approve
4. Set Up Jenkins, SonarQube & Nexus
•	Jenkins: Installed JDK 17 and Jenkins.
•	SonarQube & Nexus: Installed Docker and ran services: 
•	sudo usermod -aG docker ubuntu
•	newgrp docker
•	docker run -d -p 9000:9000 sonarqube
•	docker run -d -p 8081:8081 nexus
•	Jenkins runs on port 8080.
5. Install Required Tools
•	Installed Trivy and Kubectl on Jenkins: 
•	sudo apt install -y kubectl
•	Connected Jenkins to EKS Cluster: 
•	aws eks --region ap-south-1 update-kubeconfig --name devopsshack-cluster
6. Configure Kubernetes Resources
•	Created a namespace for the application: 
•	kubectl create namespace webapps
•	Defined service account, role, and role bindings in sa.yml, role.yml, rolebind.yml.
•	Generated a Kubernetes token and saved it in Jenkins credentials.
7. Configure Jenkins for CI/CD
•	Installed required plugins: 
o	SonarQube Scanner, Pipeline Maven Integration, Docker Pipeline, Kubernetes CLI, Credentials API
•	Set up tools: 
o	Maven (maven3), JDK
•	Added credentials: 
o	SonarQube Server, GitHub PAT, Docker credentials
8. Configure Pipeline & Deploy Application
•	Defined Maven settings in pom.xml.
•	Created a Jenkins Pipeline Job, keeping max builds 2.
•	The pipeline script is stored in Jenkinsfile.
•	Ran the build and checked pod status: 
•	kubectl get all -n webapps
•	Copied the external IP from kubectl get svc and accessed the application in the browser.
________________________________________
Troubleshooting & Optimizations
Common Issues & Fixes
Docker Login Issues
•	Issue: Jenkins failed to log in to Docker Hub.
•	Fix: Generated a Docker Hub token and stored it as a Secret Text credential in Jenkins.
Permission Issues While Building Docker Image
•	Issue: "Permission denied while trying to connect to the Docker daemon socket."
•	Fix: 
•	sudo usermod -aG docker jenkins
•	sudo systemctl restart jenkins
Kubernetes Pods Stuck in Pending State
•	Issue: No available nodes.
•	Fix: Checked worker nodes and created Node Groups in AWS EKS Console.
Incorrect Kubeadm Join Command
•	Issue: Missing port in the apiServerEndpoint.
•	Fix: Ensured proper port was provided.
Deployment Fails Due to Missing Node Group
•	Issue: No worker nodes in the cluster.
•	Fix: Created a Managed Node Group in AWS EKS.
________________________________________
Jenkins Pipeline Configuration
pipeline {
    agent any
    tools { maven 'maven3' }
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose environment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch Blue/Green')
    }
    environment {
        DOCKER_USER = 'bhargavibindu'
        IMAGE_NAME = "bhargavibindu/bankapp"
        TAG = "${params.DEPLOY_ENV}"
        KUBE_NAMESPACE = 'webapps'
        SCANNER_HOME= tool 'sonar-scanner'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/bhargavi-bindu/Blue-Green-Deployment.git'
            }
        }
        stage('Compile') { steps { sh "mvn compile" } }
        stage('Tests') { steps { sh "mvn test -DskipTests=true" } }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=Multitier -Dsonar.projectName=Multitier"
                }
            }
        }
        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker-token', variable: 'DOCKER_PASS')]) {
                        sh 'echo $DOCKER_PASS | docker login -u "bhargavibindu" --password-stdin'
                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }
    }
}
________________________________________
Conclusion
This Blue-Green Deployment setup ensures zero-downtime updates with Kubernetes and Jenkins automation. The pipeline efficiently manages CI/CD, security scanning, and smooth traffic switching between environments.


Screenshots: 
 

 

 

 

