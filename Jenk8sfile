pipeline {
    agent any
    
    tools{
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment{
        SCANNER_HOME= tool 'sonar-scanner'
        PATH = "$PATH:/usr/local/bin/aws"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Sabag10x/k8s-poc.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }
        
        stage('Unit - Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }        
        
        stage('Sonarqube - Scanner') {
            steps {
                withSonarQubeEnv('sonar'){
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.ProjectName=K8sPoc -Dsonar.ProjectKey=K8sPoc \
                    -Dsonar.java.binaries=. '''
                    
                }
            }
        }
        
        stage('Sonar-Quality-Gate') {
            steps {
                script{
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean install'
            }
        }
        
        stage('Store Artifacts in S3') {
            steps {
                script {
                    withAWS(credentials: 's3-artifact', region: 'us-east-1') {
                        sh 'aws s3 cp ./target/*.jar  s3://poc-artifacts-ipsy/Artifacts/'
                    }
                }
            }
        }    
        
        stage('Build & Tag Docker Image ') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh 'docker build -t sabag10x/k8-spoc:v2 .'
                    }
                }
            }
        }
        
        stage('Docker-Image-Scan') {
            steps {
                sh "trivy image --format table -o trivy-fs-report.html sabag10x/k8-spoc:latest"
                
            }
        } 
        
        stage('Docker-Push Image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh 'docker push sabag10x/k8-spoc:v2'
                    }
                }
            }
        }
        
        stage ('eks-deployment'){
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'k8spoc', contextName: '', credentialsId: 'kube-id', namespace: 'webapps', serverUrl: 'https://B519352EA84C7E41F780ACAD7AEBF5E2.gr7.us-east-2.eks.amazonaws.com']]) {
                        sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

                stage('Verify the Deployment') {
            steps {
               withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'k8spoc', contextName: '', credentialsId: 'kube-id', namespace: 'webapps', serverUrl: 'https://B519352EA84C7E41F780ACAD7AEBF5E2.gr7.us-east-2.eks.amazonaws.com']]) {
                        sh "kubectl get pods -n webapps"
                        sh "kubectl get svc -n webapps"
                }
            }
        }
    }
    post {
    always {
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'sabareeswaran2023@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}

}
