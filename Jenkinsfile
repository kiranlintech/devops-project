pipeline {

    agent any

    parameters {
        choice(
            name: 'DEPLOY_TARGET',
            choices: ['homelab', 'vps'],
            description: 'Select deployment target'
        )
    }

    environment {
        IMAGE_NAME = "kiranlintech/flask-devops"
        HOMELAB_HOST = "192.168.5.9"
        VPS_HOST = "213.210.37.106"
    }

    stages {

        stage('Pull Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/kiranlintech/devops-project.git'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck(
                    additionalArguments: '--scan ./',
                    odcInstallation: 'OWASP-Dependency-Check'
                )
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {

                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv('sonarqube') {

                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=flask-app \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://sonarqube:9000
                        """
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$BUILD_NUMBER .'
            }
        }

        stage('Trivy Scan') {
            steps {
                sh 'trivy image $IMAGE_NAME:$BUILD_NUMBER'
            }
        }

        stage('Docker Push') {
            steps {
                script {

                    withDockerRegistry(
                        [
                            credentialsId: 'dockerhub',
                            url: 'https://index.docker.io/v1/'
                        ]
                    ) {

                        sh """
                        docker push $IMAGE_NAME:$BUILD_NUMBER
                        docker tag $IMAGE_NAME:$BUILD_NUMBER $IMAGE_NAME:latest
                        docker push $IMAGE_NAME:latest
                        """
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {

                    def TARGET_HOST = ""

                    if (params.DEPLOY_TARGET == "homelab") {
                        TARGET_HOST = env.HOMELAB_HOST
                    } else {
                        TARGET_HOST = env.VPS_HOST
                    }

                    sh """
                    ssh ubuntu@${TARGET_HOST} '
                    docker pull ${IMAGE_NAME}:${BUILD_NUMBER}

                    docker stop flask-app || true
                    docker rm flask-app || true

                    docker run -d \
                      --name flask-app \
                      -p 8083:5000 \
                      --restart unless-stopped \
                      ${IMAGE_NAME}:${BUILD_NUMBER}
                    '
                    """
                }
            }
        }
    }

    post {

        success {
            echo "Successfully deployed to ${params.DEPLOY_TARGET}"
        }

        failure {
            echo 'Pipeline failed!'
        }
    }
}
