# DevOps Project: Terraform for EC2 provisioning & Jenkins for integration

## Introduction
This is a project for demonstrating the integration of a ReactJs project using Terraform for EC2 provisioning and Jenkins for CICD. The project is using Terraform for EC2 provisioning using AWS IAM credentials and setting up Jenkins and SonarQube withing the EC2 instance, the CICD pipeline demonstrates multiple stages for security practices and finally deploying the application as a container within the same instance.

### Architecture
1. Terraform for EC2 provisioning
2. Jenkins for CICD
3. SonarQube for static code analysis
4. Trivy for file scan and image scan
5. Docker for containerization and dockerHub for image registry

### Flow of events
1. Initially the EC2 instance and a security group required for the same is provisioned using terraform with a IAM credentials stored in the local AWS CLI configuration.
2. While provisioning, we use the ```user_data``` to running the shell script from install_jenkins.sh file in the same folder which is will spin up a Jenkins server and SonarQube server
```
resource "aws_security_group" "Jenkins-sg" {
  name        = "Jenkins-Security Group"
  description = "Open 22,443,80,8080,9000"

  # Define a single ingress rule to allow traffic on all specified ports
  ingress = [
    for port in [22, 80, 443, 8080, 9000, 3000] : {
      description      = "TLS from VPC"
      from_port        = port
      to_port          = port
      protocol         = "tcp"
      cidr_blocks      = ["0.0.0.0/0"]
      ipv6_cidr_blocks = []
      prefix_list_ids  = []
      security_groups  = []
      self             = false
    }
  ]

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Jenkins-sg"
  }
}


resource "aws_instance" "web" {
  ami                    = "ami-0f5ee92e2d63afc18"
  instance_type          = "t2.large"
  key_name               = "DevOps"
  vpc_security_group_ids = [aws_security_group.Jenkins-sg.id]
  user_data              = templatefile("./install_jenkins.sh", {})

  tags = {
    Name = "Jenkins-sonar"
  }
  root_block_device {
    volume_size = 30
  }
}
```
3. The Jenkins server is responsible for running the following stages of operations:
    1. Static code analysis using SonarQube with [quality gates](https://docs.sonarsource.com/sonarqube/latest/user-guide/quality-gates/).
    2. OWASP Dependency checks and Trivy file system scans
    3. Once all the security operations are done we can build our docker image and push it to the docker registry
    4. After pushing the image to the registry, we run Trivy image scan on the image for finding any security issues.
    5. Finally we run the containers within the instance in the last stage of our pipeline
4. Jenkinsfile:
```
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool "sonar-scanner"
    }
    stages {
        stage('CW') {
            steps {
                cleanWs()
            }
        }
        stage ('GIT CO') {
            steps {
                git branch: 'main', url: 'https://github.com/KingKong26/amazon-fe.git'
            }
        }
        stage ('SONAR CA') {
            steps {
                script {
                    withSonarQubeEnv('sonar-server') {
                        sh ''' 
                        $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=amazon \
                        -Dsonar.projectKey=amazon '''
                    }
                }
            }
        }
        stage ('SONAR QG') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage ('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage ('TRIVY FS') {
            steps {
                sh 'trivy fs . > TRIVYFS.txt'
            }
        }
        stage ('DB & DP') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                        docker build -t amazon .
                        docker tag amazon vishnumohanan/amazon:latest
                        docker push vishnumohanan/amazon:latest
                        '''
                    }
                }
            }
        }
        stage ('TRIVY IMAGE') {
            steps {
                sh 'trivy image vishnumohanan/amazon:latest > TRIVYIMAGE.txt'
            }
        }
        stage ('AMAZON APP') {
            steps {
                sh 'docker stop amazon && docker rm amazon'
                sh 'docker run -d --name amazon -p 3000:3000 vishnumohanan/amazon:latest'
            }
        }
    }
}
```
5. After completion of the project, to save server cost destroy the resources provisioned by terraform using the following command: ```terraform destroy```

### Scope for improvement
1. Branching strategy: for this demo I've kept it simple and used the main branch for everything. As the application scales you can use different branches for different purposes like tests, development, releases etc.
2. Pipeline automation: every time a new change is made we have to trigger the pipeline manually, the pipeline trigger can be automated with the help of webhooks.
3. Isolated Agent Nodes for running builds: we are running our builds in the same node as the master controller, which is [not recommended](https://dspenard.medium.com/ease-your-jenkins-master-node-pains-with-remote-agents-9fc6c2e336ee), the recommended way of doing this is using [agent nodes](https://www.jenkins.io/doc/book/using/using-agents/)
4. Remote store for Terraform state files: in the current setup I have kept the state files local and ignored, but and as we move to a more sophisticated production operations we should consider moving the TF state files into a safe and remote store like s3 or HashiCorp Vault for security, collaboration and more.

## Summary
The goal of this project was to understand IaC concepts and pipeline formation using Jenkins. I got to explore various nuances and concerns to look into while using terraform, such as safe keeping of state files and credentials. And I also discovered packages like Trivy and OWASP for implementation of DevSecOps practices.

# NOTE: The resources are not available to save the server cost, as the goal of this demo is to learn these topics rather than running the application in live.

