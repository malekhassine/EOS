
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
	TIMEOUT_VALUE = '600m'
        DOCKERHUB_USERNAME = "malekhassine"
        // Ensure Docker credentials are stored securely in Jenkins
	MASTER_NODE = 'https://192.168.63.136:6443'
        KUBE_CREDENTIALS_ID = 'tokemaster2'
        //REMOTE_USER = 'master'       // SSH username on the master node
        //REMOTE_HOST = '192.168.63.133'  // IP or hostname of the master node
        //SSH_CREDENTIALS_ID = 'master-ssh' // ID of the SS
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
	     
        stage('Update Trivy Database') {
            steps {
                script {
                    // Clear the Trivy database cache to ensure the most current data is used
                    sh 'docker run --rm -v $PWD:/tmp/.cache/ aquasec/trivy --reset'
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
                            aquasec/trivy image --reset \
                            --scanners vuln --severity CRITICAL,HIGH,MEDIUM \
                            --timeout ${TIMEOUT_VALUE} \
                            ${imageTag} > ${trivyReportFile}
                        """

                        // Archive Trivy reports for all microservices in a dedicated directory
                        archiveArtifacts artifacts: trivyReportFile, allowEmptyArchive: true
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
        script {
	
            // Define the Kubernetes credentials ID
            def kubeCredentialsId = 'tokemaster2'
            
            // Use withCredentials to securely access Kubernetes token
            withCredentials([string(credentialsId: kubeCredentialsId, variable: 'KUBE_TOKEN')]) {
                // Define the kubectl base command with token
                def kubectlBaseCmd = "./kubectl --token=${KUBE_TOKEN}"
                
                // Download kubectl v1.30.1
                sh 'curl -LO https://dl.k8s.io/release/v1.30.1/bin/linux/amd64/kubectl'
                sh 'chmod u+x ./kubectl'
		 def deployenv = ''
                // Use the downloaded kubectl for deployment
                if (env.BRANCH_NAME == 'test') {
                     deployenv = 'test'
                } else if (env.BRANCH_NAME == 'master') {
                    deployenv = 'prod'
                }
		    sh "rm -f deploy_to_${deployenv}.sh"
                        sh "wget \"https://raw.githubusercontent.com/malekhassine/EOS/test/deploy_to_test.sh\""
                        sh "scp deploy_to_${deployenv}.sh $MASTER_NODE:~"
                        sh "${kubectlBaseCmd} --server=$MASTER_NODE chmod +x deploy_to_${deployenv}.sh"
                        sh "${kubectlBaseCmd} --server=$MASTER_NODE ./deploy_to_${deployenv}.sh"
                        sh "${kubectlBaseCmd} --server=$MASTER_NODE kubectl apply -f ${deployenv}_manifests/namespace.yml"
                        sh "${kubectlBaseCmd} --server=$MASTER_NODE kubectl apply -f ${deployenv}_manifests/infrastructure/"
                        for (def service in services) {
                            sh "ssh $MASTER_NODE kubectl apply -f ${deployenv}_manifests/microservices/${service}.yml"
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




        
    
