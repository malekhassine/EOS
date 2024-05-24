
def microservices = ['ecomm-cart']

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
        DOCKERHUB_USERNAME = "malekhassine"
        // Ensure Docker credentials are stored securely in Jenkins
	MASTER_NODE = 'https://192.168.63.133:6443'
        KUBE_CREDENTIALS_ID = 'tokenmaster'
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

  stage('Trivy Image Scan') {
            when {
                expression { (env.BRANCH_NAME == 'dev') || (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                script {
                    // Scan each Docker image for vulnerabilities using Trivy
                    for (def service in microservices) {
                        def trivyReportFile = "trivy-${service}.txt"

                    // Combine vulnerability and severity filters for clarity and flexibility
                        def trivyScanArgs = "--scanners vuln --severiCRITICAL,HIGH,MEDIUM--timeout 30m"
 if (env.BRANCH_NAME == 'test') {
                            sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $PWD:/tmp/.cache/ aquasec/trivy image --scanners vuln --timeout 30m ${DOCKERHUB_USERNAME}/${service}_test:latest > ${trivyReportFile}"
                        } else if (env.BRANCH_NAME == 'master') {
                            sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $PWD:/tmp/.cache/ aquasec/trivy image --scanners vuln --timeout 30m ${DOCKERHUB_USERNAME}/${service}_prod:latest > ${trivyReportFile}"
                        } else if (env.BRANCH_NAME == 'dev') {
                            sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v $PWD:/tmp/.cache/ aquasec/trivy image --scanners vuln --timeout 30m ${DOCKERHUB_USERNAME}/${service}_dev:latest > ${trivyReportFile}"
                        }
                         // Archive Trivy reports for all microservices in a dedicated directory
                        archiveArtifacts "**/*.txt"
                    }
                }
            }
        }


        stage('Docker Push') {
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
	}
        stage('Deploy to Kubernetes') {
            when {
                expression { (env.BRANCH_NAME == 'test') || (env.BRANCH_NAME == 'master') }
            }
            steps {
                withCredentials([string(credentialsId: env.KUBE_CREDENTIALS_ID, variable: 'KUBE_TOKEN')]) {

                script {
	           
                     if (env.BRANCH_NAME == 'test') {
                            sh "kubectl --token=$KUBE_TOKEN --server=$MASTER_NODE apply -f cart.yml"
                        sh "kubectl --token=$KUBE_TOKEN --server=$MASTER_NODE apply -f namespace.yml" }

                   else if (env.BRANCH_NAME == 'master') {
                            sh "kubectl --token=$KUBE_TOKEN --server=$MASTER_NODE apply -f cart.yml"
                            sh "kubectl --token=$KUBE_TOKEN --server=$MASTER_NODE apply -f namespace.yml"                       
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




        }
    
