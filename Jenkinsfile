// CI Jenkinsfile for GCBank Application

pipeline   {
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
                    sh ''' ${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectName=Project-Bank \
                            -Dsonar.projectKey=Project-Bank \
                            -Dsonar.java.binaries=target '''
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
                withMaven(globalMavenSettingsConfig: 'devopsshack', mavenName: 'maven3', mavenSettingConfig: '', trackerName: 'maven') {
                    sh 'mvn deploy'
                }
                
            }
        }
// here we have to add the docker credentials in Jenkins credentials
        stage('Build & Tag Docker Image') {
            steps {
                withDockerRegistry(credentialsId: 'docker-creds') {
                    sh 'docker build -t sathish103/project:$IMAGE_TAG .'
                } 

            } 
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --fromat table -o image-report.html sathish103/project:$IMAGE_TAG'
            }
        }

        stage('Docker Push Image') {
            steps {
                withDockerRegistry(credentialsId: 'docker-creds') {
                    sh 'docker push sathish103/project:$IMAGE_TAG'
                } 

            } 
        }

     /*   stage('Update Manifest file with image tag') {
            steps {
                script  {
                    // clean workspace before updating the manifest file
                    cleanWs()
                    withCredentials([usernamePassword(credentialsId: 'Git-creds', passwordVariable: '', usernameVariable: '')])                        
                    sh '''
                        # Clone the project repository
                        git clone https://github.com/sathish103/Multi-Tier-BankApp-CI.git

                        # Update the manifest.yaml with the new image tag
                        cd manifest-cd/
                        sed -i 's|image: devopsshack/gcbank:.*|image: sathish103/project:$IMAGE_TAG|' k8s/deployment.yaml

                        # Confirm the changes
                        echo "Updates manifest file content:"
                        cat manifest-cd/deployment.yaml

                        # Commit and push the changes

                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@gmail.com"
                        git add manifest-cd/deployment.yaml
                        git commit -m "Updated deployment.yaml with image tag $IMAGE_TAG"
                        git push origin main
                    '''
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
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: '567adddi.jais@gmail.com',
                    from: 'jenkins@devopsshack.com',
                    replyTo: 'jenkins@devopsshack.com',
                    mimeType: 'text/html',
                )
            }
        }
    }/*
}






