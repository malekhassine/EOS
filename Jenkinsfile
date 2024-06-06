
def microservices = ['ecomm-cart']
def deployenv = ''
if (env.BRANCH_NAME == 'test') {
    deployenv = 'test'
} else if (env.BRANCH_NAME == 'master') {
    deployenv = 'prod'
}

def COLOR_MAP = [
	'FAILURE' : 'danger',
	'SUCCESS' : 'good'
]

pipeline {
    agent any
    tools {
    maven 'maven'
    
}


    environment {
	TIMEOUT_VALUE = '600m'
    DOCKERHUB_USERNAME = "malekhassine"
     // Ensure Docker credentials are stored securely in Jenkins
	//MASTER_NODE = '192.168.63.137:6443'
    KUBE_CREDENTIALS_ID = 'tokemaster2'
    REMOTE_USER = 'ubuntu'       // SSH username on the master node(echo $USER)
    REMOTE_HOST = '192.168.63.137'  // IP or hostname of the master node
	SSH_CREDENTIALS_ID = 'id_rsa' // ID of the SSh rsa key
	//SSH_K8S_PROD = 'aws-ssh-id'
	K8S_EC2_USER= 'ubuntu'
	K8S_EC2_MASTER ='3.82.192.23'
    }

    stages {
        stage('Git checkout Stage') {
            steps {
                git changelog: false, poll: false, url: 'https://github.com/malekhassine/EOS'
            }
        }

        stage('Check Git Secrets') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Check each microservice for secrets
                    for (def service in microservices) {
                        dir(service) {
                            // Run TruffleHog to check for secrets in the repository
                            sh 'docker run --rm gesellix/trufflehog --json https://github.com/malekhassine/EOS.git > trufflehog.json'
                            sh 'cat trufflehog.json' // Output the results
                        }
                    }
                }
            }
        }

      

        stage('Build') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Build each microservice using Maven
                    for (def service in microservices) {
                        dir(service) {
                            sh 'mvn clean install'
                        }
                    }
                }
            }
        }

        stage('Unit Test') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Run unit tests for each microservice using Maven
                    for (def service in microservices) {
                        dir(service) {
                            sh 'mvn test'
                        }
                    }
                }
            }
        }
        stage('SonarQube Analysis and dependency check') {
               when {
                              expression {
                                  (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master')
                              }
                          }
          steps {
            script {
               // Run unit tests for each microservice using Maven
                  for (def service in microservices) {
                     dir(service) {
                         withSonarQubeEnv('sonarqube') {
                            sh 'mvn sonar:sonar'

                          }
                           dependencyCheck additionalArguments: '--format HTML', odcInstallation: 'dependency-Check'
             }
           }
          }
         }
        }

         
        stage('Docker Login') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Log into Docker Hub using Jenkins credentials
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD"
                    }
                }
            }
        }

        stage('Docker Build') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Build Docker images for each microservice based on the branch
                    for (def service in microservices) {
                        dir(service) {
                            if (env.BRANCH_NAME == 'test') {
                                sh "docker build -t ${DOCKERHUB_USERNAME}/${service}_test:latest ."
                            } else if (env.BRANCH_NAME == 'master') {
                                sh "docker build -t ${DOCKERHUB_USERNAME}/${service}_prod:latest ."
                            } else if (env.BRANCH_NAME == 'dev') {
                                sh "docker build -t ${DOCKERHUB_USERNAME}/${service}_dev:latest ."
                            }
                        }
                    }
                }
            }
        }
     /*   stage('Update Trivy Database') {
            steps {
                script {
                    // Update the Trivy database
                    sh 'docker run --rm -v $PWD:/tmp/.cache/ aquasec/trivy image --download-db-only'
                }
            }
        }

         stage('Trivy Image Scan') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Scan each Docker image for vulnerabilities using Trivy
                    for (def service : microservices) {
                        def trivyReportFile = "trivy-${service}.txt"
                        def imageTag

                        if (env.BRANCH_NAME == 'test') {
                            imageTag = "${DOCKERHUB_USERNAME}/${service}_test:latest"
                        } else if (env.BRANCH_NAME == 'master') {
                            imageTag = "${DOCKERHUB_USERNAME}/${service}_prod:latest"
                        } else if (env.BRANCH_NAME == 'dev') {
                            imageTag = "${DOCKERHUB_USERNAME}/${service}_dev:latest"
                        }

                        sh """
                            docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            -v $PWD:/tmp/.cache/ \
                            aquasec/trivy image --scanners vuln --severity CRITICAL,HIGH,MEDIUM \
                            --timeout ${TIMEOUT_VALUE} \
                            ${imageTag} > ${trivyReportFile}
                        """

                        // Archive Trivy reports for all microservices in a dedicated directory
                        archiveArtifacts artifacts: trivyReportFile, allowEmptyArchive: true
                    }
                }
            }
        }*/


       /* stage('Docker Push') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Push each Docker image to Docker Hub based on the branch
                    for (def service in microservices) {
                        if (env.BRANCH_NAME == 'test') {
                            sh "docker push ${DOCKERHUB_USERNAME}/${service}_test:latest"
                            sh "docker rmi ${DOCKERHUB_USERNAME}/${service}_test:latest"
                        } else if (env.BRANCH_NAME == 'master') {
                            sh "docker push ${DOCKERHUB_USERNAME}/${service}_prod:latest"
                            sh "docker rmi ${DOCKERHUB_USERNAME}/${service}_prod:latest"
                        } else if (env.BRANCH_NAME == 'dev') {
                            sh "docker push ${DOCKERHUB_USERNAME}/${service}_dev:latest"
                            sh "docker rmi ${DOCKERHUB_USERNAME}/${service}_dev:latest"
                        }
                    }
                }
	    }
	}*/
	     stage('Get YAML Files') {
            when {
                expression { (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    script {
                        sh "rm -f deploy_to_${deployenv}.sh"
                        sh "curl -O https://raw.githubusercontent.com/malekhassine/EOS/test/deploy_to_${deployenv}.sh "
                        sh "scp deploy_to_${deployenv}.sh $REMOTE_USER@$REMOTE_HOST:~"
                        sh "ssh $REMOTE_USER@$REMOTE_HOST chmod +x deploy_to_${deployenv}.sh"
                        sh "ssh $REMOTE_USER@$REMOTE_HOST ./deploy_to_${deployenv}.sh"
                    }
                }
            }
        }
    stage('Deploy to Kubernetes') {
            when {
                expression { (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    script{
                        sh " [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh "
		                sh " ssh-keyscan -t rsa,dsa ${REMOTE_HOST} >> ~/.ssh/known_hosts "
		                sh " ssh $REMOTE_USER@$REMOTE_HOST kubectl get nodes  "
                        sh "ssh $REMOTE_USER@$REMOTE_HOST kubectl apply -f ${deployenv}_manifests/namespace.yml"
                        sh "ssh $REMOTE_USER@$REMOTE_HOST kubectl apply -f ${deployenv}_manifests/infrastructure/"
                        for (service in services) {
                            sh "ssh $REMOTE_USER@$REMOTE_HOST kubectl apply -f ${deployenv}_manifests/microservices/${service}.yml"
                        }
                    }
                }
            }
        }



    }
}
	
post {
  // Success notification
  success {
    script {
      slackSend channel: '#dev', color: 'good', message: "Pipeline '${env.JOB_NAME}' Build Successful!"
    }
  }
  // Failure notification
  failure {
    script {
      slackSend channel: '#dev', color: 'danger', message: "Pipeline '${env.JOB_NAME}' Build Failed!"
    }
  }
}




        
    
