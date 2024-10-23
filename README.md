# Deploying-Background-Remover-Python-Flask-App-on-AWS-EKS-Using-DevSecOps-Approach

### Project Summary:

This project involves the deployment of a Prime Video clone using a full-stack serverless architecture on AWS. It utilizes several AWS services to create a scalable, high-performance streaming platform that mimics the functionality of Amazon's Prime Video.

The architecture includes AWS Amplify for seamless front-end hosting and CI/CD integration, and AWS Lambda functions for handling backend logic and processing. DynamoDB is used as a NoSQL database to manage user data and video metadata efficiently, while Amazon S3 is utilized for scalable storage of video files and assets. AWS CloudFront is integrated to deliver content globally with low latency, ensuring a smooth streaming experience.

Authentication and user management are handled by Amazon Cognito, providing secure sign-in and user data management. The deployment setup uses Infrastructure as Code (IaC) principles, implemented with AWS CloudFormation templates, to automate resource provisioning, ensuring consistency and reducing manual configuration.

This project showcases a robust deployment approach for a streaming service by utilizing the capabilities of serverless computing, scalable storage, and global content delivery on AWS.


### Application Development:
The background remover is a simple Flask application using Rembg, a tool for removing image backgrounds. The project is hosted on GitHub, and the local development setup includes Jenkins for CI/CD.

### CI/CD Pipeline (Jenkins):
The Jenkins pipeline performs several tasks:
- Cleans the workspace.
- Pulls the latest code from the GitHub repository.
- SonarQube is integrated for code quality analysis, using Sonar Scanner.

### Monitoring Setup:
Prometheus and Grafana are installed to monitor the application and infrastructure:
- Prometheus collects metrics and provides a dashboard for monitoring various services, including Node Exporter for system-level metrics.
- Grafana is used for visualizing these metrics. Steps include installing both tools on Linux, configuring system services, and setting up Prometheus to scrape data from Jenkins and Node Exporter.

### Kubernetes Monitoring:
Prometheus is also configured to monitor a Kubernetes cluster, using Helm to deploy Node Exporter on the nodes. Metrics are collected from the cluster, enhancing observability.

### ArgoCD Deployment:
The guide explains how to deploy applications using ArgoCD, a GitOps tool. The GitHub repository is set as the source for application manifests, and synchronization policies are configured for automated deployments to Kubernetes.

### Cleanup:
The final step involves cleaning up resources, including terminating EC2 instances, EKS clusters, and associated resources on AWS.


## Jenkins Pipeline Script:

```
pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage ("Clean workspace") {
            steps {
                cleanWs()
            }
        }
        stage ("Git checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/yeshwanthlm/background-remover-python-app.git'
            }
        }
        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=background-remover-python-app \
                    -Dsonar.projectKey=background-remover-python-app '''
                }
            }
        }
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage ("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivy.txt"
            }
        }
        stage ("Build Docker Image") {
            steps {
                sh "docker build -t background-remover-python-app ."
            }
        }
        stage ("Tag & Push to DockerHub") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker tag background-remover-python-app amonkincloud/background-remover-python-app:latest"
                        sh "docker push amonkincloud/background-remover-python-app:latest"
                    }
                }
            }
        }
        stage('Docker Scout Image') {
            steps {
                script {
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                       sh 'docker-scout quickview amonkincloud/background-remover-python-app:latest'
                       sh 'docker-scout cves amonkincloud/background-remover-python-app:latest'
                       sh 'docker-scout recommendations amonkincloud/background-remover-python-app:latest'
                   }
                }
            }
        }
        stage ("Deploy to Container") {
            steps {
                sh 'docker run -d --name background-remover-python-app -p 5100:5100 amonkincloud/background-remover-python-app:latest'
            }
        }
    }
    post {
    always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: """
                <html>
                <body>
                    <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                    </div>
                    <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                    </div>
                    <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                    </div>
                </body>
                </html>
            """,
            to: 'provide_your_Email_id_here',
            mimeType: 'text/html',
            attachmentsPattern: 'trivy.txt'
        }
    }
}

```

