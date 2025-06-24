pipeline {
    agent { label 'build-agent' }

    environment {
        REPO_URL = 'https://github.com/Thirumales/jenkins-ci-cd-workflow.git'
        IMAGE_NAME = 'api_go_jenkins_demo'
        REGISTRY_URL = 'image-registry-harbor-registry.iks-get.harbor.com'
        PROJECT_NAME = 'library'
        CA_CERT_PATH = '/usr/local/share/ca-certificates/ca.pem'
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        BUILD_TAG = "testing${BUILD_NUMBER}"
    }

    stages {
        stage('Docker Login') {
            steps {
                container('docker') {
                    script {
                        echo 'Logging into Docker Hub...'
                        withCredentials([usernamePassword(credentialsId: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USER', passwordVariable: 'DOCKER_HUB_PASS')]) {
                            sh 'docker login -u $DOCKER_HUB_USER -p $DOCKER_HUB_PASS'
                        }
                    }
                }
            }
        }

        stage('Checkout') {
            steps {
                container('jnlp') {
                    script {
                        echo 'Checking out code...'
                        sh 'git config --global http.sslVerify false'
                        git branch: 'main', url: "${REPO_URL}"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    script {
                        echo 'Building Docker image...'
                        sh 'docker build -t ${REGISTRY_URL}/${PROJECT_NAME}/${IMAGE_NAME}:${BUILD_TAG} .'
                    }
                }
            }
        }

        stage('Run Unit Tests') {
            steps {
                container('golang') {
                    script {
                        echo 'Running unit tests...'
                        sh 'go test ./...'
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    script {
                        sh 'mkdir -p /etc/docker/certs.d/${REGISTRY_URL}'
                        sh 'cp ${CA_CERT_PATH} /etc/docker/certs.d/${REGISTRY_URL}/${REGISTRY_URL}.crt'
                        echo 'Pushing Docker image to Harbor...'
                        withCredentials([usernamePassword(credentialsId: 'Harbor_Password', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                            sh 'docker login -u $DOCKER_USER -p $DOCKER_PASS ${REGISTRY_URL}'
                        }
                        sh 'docker push ${REGISTRY_URL}/${PROJECT_NAME}/${IMAGE_NAME}:${BUILD_TAG}'
                    }
                }
            }
        }

        stage('Deploy App to Kubernetes') {
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        script {
                            echo 'Updating Kubernetes manifest with image tag...'
                            sh 'sed -i "s|<TAG>|${BUILD_TAG}|" web-app.yaml'
                            echo 'Applying Kubernetes manifest...'
                            sh 'kubectl apply -f web-app.yaml'
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully. Image pushed and deployment applied.'
        }
        failure {
            echo 'Pipeline failed. Please check the logs.'
        }
    }
}


