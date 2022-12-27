pipeline{
    agent any 
    environment{
        VERSION = "${env.BUILD_ID}"
    }
    stages{
        stage("sonar quality check"){
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
                                docker build -t 44.198.51.15:8083/springapp:${VERSION} .
                                docker login -u admin -p $docker_password 44.198.51.15:8083 
                                docker push  44.198.51.15:8083/springapp:${VERSION}
                                docker rmi 44.198.51.15:8083/springapp:${VERSION}
                            '''
                    }
                }
            }
        }
        stage('indentifying misconfigs using datree in helm charts'){
            steps{
                script{
                    dir('kubernetes/') {
                              sh 'helm datree test myapp/'
                        }
                    }
                }
            }
    }
}
