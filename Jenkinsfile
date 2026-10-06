pipeline {
    agent { label 'Jenkins-Agent' }
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "ilsamukhtar"
        DOCKER_PASS = 'dockerhub'
        IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        CD_JENKINS_URL = "http://<APNA-JENKINS-ADDRESS>:8080"
    }

    stages {

        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/ilsamukhtar/register-app'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            steps {
                sh "mvn test"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                        sh "mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar"
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }

                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh "docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${IMAGE_NAME}:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table"
                }
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'API_TOKEN')]) {
                    sh '''
                        curl -s --user clouduser:$API_TOKEN -X POST \
                          -H 'cache-control: no-cache' \
                          -H 'content-type: application/x-www-form-urlencoded' \
                          --data "IMAGE_TAG=${IMAGE_TAG}" \
                          "${CD_JENKINS_URL}/job/gitops-register-app-cd/buildWithParameters?token=gitops-token"
                    '''
                }
            }
        }
    }

    post {
        failure {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                     subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed",
                     mimeType: 'text/html',
                     to: "ilsamukhtar3@gmail.com"
        }
        success {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                     subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful",
                     mimeType: 'text/html',
                     to: "ilsamukhtar3@gmail.com"
        }
    }
}
