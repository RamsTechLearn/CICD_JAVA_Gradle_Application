pipeline{
    agent any 
    environment{
        VERSION = "${env.BUILD_ID}"
    }
    stages{
        stage("sonar quality check"){
            agent {
                docker {
                    image 'openjdk:11'
                }
            }
            steps{
                script{
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                            sh 'chmod +x gradlew'
                            sh './gradlew sonarqube'
                    }

                    timeout(time: 1, unit: 'HOURS') {
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
                    withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                             sh '''
                                docker build -t 54.85.255.77:8083/springapp:${VERSION} .
                                docker login -u admin -p $docker_password 54.85.255.77:8083 
                                docker push  54.85.255.77:8083/springapp:${VERSION}
                                docker rmi 54.85.255.77:8083/springapp:${VERSION}
                                docker login -u kcrstechlearn -p Faber@0724nikon
                                ID=$(docker build -q -t creack/node .)
                                docker tag $ID kcrstechlearn/springapp:${VERSION}
                                docker push kcrstechlearn/springapp:${VERSION}
                                docker rmi kcrstechlearn/springapp:${VERSION}
                            '''
                    }
                }
            }
        }
        stage('indentifying misconfigs using datree in helm charts'){
            steps{
                script{

                    dir('kubernetes/') {
                        withEnv(['DATREE_TOKEN=b53f8a83-6947-4487-85e7-e950b0fabfdd']) {
                              sh 'helm datree test myapp/'
                        }
                    }
                }
            }
        }
        stage("pushing the helm charts to nexus"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                          dir('kubernetes/') {
                             sh '''
                                 helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                                 tar -czvf  myapp-${helmversion}.tgz myapp/
                                 curl -u admin:$docker_password http://54.85.255.77:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                            '''
                          }
                    }
                }
            }
        }
        stage('Deploying application on k8s cluster') {
            steps {
               script{
                    withCredentials([file(credentialsId: 'kubernetes-config', variable: 'KUBECONFIG')]) {
                        dir('kubernetes/') {
                          sh 'helm upgrade --install --set image.repository="kcrstechlearn/springapp" --set image.tag="${VERSION}" myjavaapp myapp/ ' 
                        }
                    }
                }
            }
        }
        stage('verifying app deployment'){
            steps{
                script{
                     withCredentials([File(credentialsId: 'kubernetes-config', variable: 'KUBECONFIG')]) {
                         sh 'kubectl run curl --image=curlimages/curl -i --rm --restart=Never -- curl myjavaapp-myapp:8080'

                     }
                }
            }
        }

    }
}