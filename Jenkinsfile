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
