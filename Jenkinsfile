pipeline{
    agent any
    environment{
        VERSION = "${env.BUILD_ID}"
    }
    stages{
        stage("sonar quality check"){
            agent{
                docker {
                    image 'openjdk:11'
                }
            }
            steps{
                script{
                    withSonarQubeEnv(credentialsId: 'sonarqube-token') {
                            sh 'chmod +x gradlew'
                            sh './gradlew sonarqube'
                    }

                    timeout(5) {
                      def qg = waitForQualityGate()
                      if (qg.status != 'OK') {
                           error "Pipeline aborted due to quality gate failure: ${qg.status}"
                      }
                    }

                }
            }
        }
        stage("docker build & docker push"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'nexus_pass', variable: 'nexus_password')]) {
                             sh '''
                                docker build -t 3.220.243.252:8083/springapp:${VERSION} .
                                docker login -u admin -p $nexus_password 3.220.243.252:8083
                                docker push 3.220.243.252:8083/springapp:${VERSION}
								docker logout
                            '''
				    }
				    withCredentials([string(credentialsId: 'dockerhub_pass', variable: 'dockerhub_password')]) {
					         sh '''
							    docker tag 3.220.243.252:8083/springapp:${VERSION} kishanth1994/springapp:${VERSION}
								docker login -u kishanth1994 -p $dockerhub_pass
								docker push kishanth1994/springapp:${VERSION}
								docker rmi 3.220.243.252:8083/springapp:${VERSION}
								docker rmi kishanth1994/springapp:${VERSION}
							'''
                    }
                }
            }
        }
        stage("Identifying misconfigurations using datree in helm charts"){
            steps{
                script{

                    dir('kubernetes/') {
                        withEnv(['DATREE_TOKEN=87nXtFQA2QxJ1DCuziufez']) {
                              sh 'helm datree test myapp/'
                        }    
                    }
                }
            }
        }
        stage("Pushing the helm charts to nexus"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'nexus_pass', variable: 'nexus_password')]) {
                          dir('kubernetes/') {
                             sh '''
                                  helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                                  tar -czvf myapp-${helmversion}.tgz myapp/
                                  curl -u admin:$nexus_password http://3.220.243.252:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                            '''
                          }
                    }
                }
            }
        }
        stage("Copying the playbook and kubernetes manifests to Ansible server"){
		    steps{
			    script{
				    echo "${WORKSPACE}"
					        sh '''
							     sudo scp -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa ${WORKSPACE}/kubernetes/manifests-yamls/*.yaml root@172.31.85.166:/etc/ansible/kubernetes/
							'''     
				}
            }
        }			
							
        stage('manual approval'){
            steps{
                script{
                    timeout(10) {
                        mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Go to build url and approve the deployment request <br> URL of the build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "devopsatitspeak@gmail.com";  
                        input(id: "Deploy Gate", message: "Deploy ${params.project_name}?", ok: 'Deploy')
                    }
                }
            }
        }

        stage('Deploying application on k8s cluster from ansible-server using playbook') {
            steps {
               script{
			        sh 'ssh -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa root@172.31.85.166 "kubectl config get-contexts; kubectl config use-context kubernetes-admin@kubernetes; whoami && hostname; ansible-playbook -i /etc/ansible/kubernetes/hosts /etc/ansible/kubernetes/playbook-deployment-service.yaml; sudo rm -rf /etc/ansible/kubernetes/*.yaml;"'   
               }
            }
        }
        
        stage('verifying app deployment'){
            steps{
                script{
                     withCredentials([kubeconfigFile(credentialsId: 'kubernetes-configs', variable: 'KUBECONFIG')]) {
                         sh 'kubectl run curl --image=curlimages/curl -i --rm --restart=Never -- curl myjavaapp-myapp:8080'

                     }
                }
            }
        }
    }
	   
    post {
		always {
			mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL of the build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "devopsatitspeak@gmail.com";  
		 }
	   }
}
