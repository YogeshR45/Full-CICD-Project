pipeline {
    agent any

    environment {
        DOCKER_CREDS    = credentials('dockerhub-creds')
        KUBECONFIG_DATA = credentials('k3s-config')
        IMAGE_NAME      = "yogeshdock45/drepo"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/YogeshR45/Full-CICD-Project.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh """
                  docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                """
            }
        }
     stage('Login to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh """
                        echo "$PASS" | docker login -u "$USER" --password-stdin
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                  docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                """
            }
        }
     stage('Update YAML With New Image') {
            steps {
                sh """
                  sed -i 's|image:.*|image: ${IMAGE_NAME}:${BUILD_NUMBER}|g' deployment.yaml
                """
            }
        }

        stage('Deploy to K3s') {
            steps {
                withCredentials([file(credentialsId: 'k3s-config', variable: 'KCFG')]) {
                    sh """
                        export KUBECONFIG=$KCFG
                        kubectl apply -f deployment.yaml
                        kubectl apply -f service.yaml
                    """
                }
            }
        }
    }
}

