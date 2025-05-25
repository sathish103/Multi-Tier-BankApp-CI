// CI Jenkinsfile for GCBank Application

pipeline {
    agent any
    tools {
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }
// Add git creds to the jenkins credentials
    stages {
        stage('Git Checkout') {
            steps {
                 git branch: 'main', credentialsId: 'Git-creds', url: 'https://github.com/sathish103/Multi-Tier-BankApp-CI.git'
            }            
        }
        
        stage('compile') {
            steps {               
                sh 'mvn compile'
            }
        }
        stage('Test') {
            steps {                
                sh 'mvn test'
            }
        }
        
        stage('Trivy FS scan'){
            steps {                
                sh 'trivy fs --format table -o fs-report.html .'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {               
                withSonarQubeEnv('SonarQube') {
                    sh '''${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=Project-Bank \
                        -Dsonar.projectName=Project-Bank \
                        -Dsonar.java.binaries=target'''
                }
            }
        }
//setup sonarqube-webhook for jenkins
        stage('Quality Gate check') { 
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Build') {
            steps{
                sh 'mvn package'
            }
        } 
//here we have to add the nexus repos "mvnrelease" and "snapshot" URL to the pom.xml and
// update the nexus repo ID & creds in Jenkins "managed files" settings.xml
        stage('Publish Artifacts') {
            steps {
                withMaven(globalMavenSettingsConfig: 'devopsshack', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh 'mvn deploy'
                }
                
            }
        }
//here we have to add the docker credentials in Jenkins credentials
        stage('Build & Tag Docker Image') {
            steps {
                withDockerRegistry(credentialsId: 'docker-creds', url: 'https://index.docker.io/v1/') {
                    sh 'docker build -t sathish103/project:$IMAGE_TAG .'
                } 

            } 
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --format table -o image-report.html sathish103/project:$IMAGE_TAG'
            }
        }

        stage('Docker Push Image') {
            steps {
                withDockerRegistry(credentialsId: 'docker-creds', url: 'https://index.docker.io/v1/') {
                    sh 'docker push sathish103/project:$IMAGE_TAG'
                } 

            } 
        }

        stage('Update Manifest file with image tag') {
            steps {
                script {
                    cleanWs()

                    withCredentials([usernamePassword(credentialsId: 'Git-creds', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh '''
                            # Clone the repo using HTTPS with credentials
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/sathish103/Multi-Tier-BankApp-CI.git

                            cd Multi-Tier-BankApp-CI

                            # Update the manifest file with the new image tag
                            sed -i "s|image: sathish103/project:.*|image: sathish103/project:${IMAGE_TAG}|" manifest-cd/k8s/deployment.yaml

                            echo "Updated manifest file content:"
                            cat manifest-cd/k8s/deployment.yaml

                            # Configure Git user
                            git config --global user.name "sathish103" 
                            git config --global user.email "sathishreddys786@gmail.com"
                           
                            # Commit and push the change
                            git add manifest-cd/k8s/deployment.yaml
                            git commit -m "Updated deployment.yaml with image tag ${IMAGE_TAG}"
                            git push origin main
                        '''
                    }
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
                </body>
                </html>
                """
                emailext (
                    subject: "${jobName} - Build #${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'sathiscool444@gmail.com',
                    from: 'sathishreddys786@gmail.com',
                    replyTo: 'sathishreddys786@gmail.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}

