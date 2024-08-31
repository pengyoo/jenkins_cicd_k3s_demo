pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-hub-registry'
        DOCKER_IMAGE = 'ypydd88/demo'
        K8S_CREDENTIALS_ID = 'k3s'
        K8S_NAMESPACE = 'demo'
        DOCKER_PATH = '/usr/bin/docker'
        KUBECTL_PATH = '/usr/local/bin/kubectl'
    }

    tools {
        maven 'Maven3.9.9'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'http://172.22.144.2:3000/ypydd88/demo.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean -DskipTests package'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "${DOCKER_PATH} build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "${DOCKER_PATH} login -u ${DOCKER_USER} -p ${DOCKER_PASS}"
                    sh "${DOCKER_PATH} push ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                    sh "${DOCKER_PATH} tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest"
                    sh "${DOCKER_PATH} push ${DOCKER_IMAGE}:latest"
                }
            }
        }

        stage('Deploy to K3s') {
            steps {
                withKubeConfig([credentialsId: "${K8S_CREDENTIALS_ID}", namespace: "${K8S_NAMESPACE}"]) {
                    sh """
                        echo "Current working directory:"
                        pwd
                        echo "Kubectl version:"
                        ${KUBECTL_PATH} version --client
                        echo "Trying to connect to the cluster:"
                        ${KUBECTL_PATH} cluster-info
                        echo "Ensuring namespace exists:"
                        ${KUBECTL_PATH} create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | ${KUBECTL_PATH} apply -f -
                        echo "Applying Kubernetes resources:"
                        ${KUBECTL_PATH} apply -f kubernetes/
                        echo "Updating deployment image:"
                        ${KUBECTL_PATH} set image deployment/demo demo=${DOCKER_IMAGE}:${BUILD_NUMBER} -n ${K8S_NAMESPACE}
                        echo "Checking rollout status:"
                        ${KUBECTL_PATH} rollout status deployment/demo -n ${K8S_NAMESPACE}
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}