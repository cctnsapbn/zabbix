```groovy
pipeline {
    agent any

    environment {
        DOCKER_REPO    = 'dockercctns/zabbix'
        IMAGE_TAG      = "${BUILD_NUMBER}"

        DOCKER_CREDS_ID = 'dockerhub-creds'
        KUBE_CONFIG_ID  = 'k8s-kubeconfig'

        K8S_NAMESPACE  = 'zabbix'
    }

    stages {

        stage('Checkout SCM') {
            steps {
                echo 'Checking out source code from GitHub...'
                checkout scm
            }
        }

        stage('Verify Source') {
            steps {
                sh '''
                    echo "===== Repository ====="
                    ls -la

                    echo "===== Kubernetes Files ====="
                    ls -la k8s
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "===== Building Docker Image ====="

                    docker build \
                        -t ${DOCKER_REPO}:${IMAGE_TAG} \
                        -t ${DOCKER_REPO}:latest \
                        -f docker/Dockerfile .
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDS_ID}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "===== Docker Hub Login ====="

                        echo "$DOCKER_PASSWORD" | docker login \
                            --username "$DOCKER_USERNAME" \
                            --password-stdin
                    '''
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                sh '''
                    echo "===== Pushing Image ====="

                    docker push ${DOCKER_REPO}:${IMAGE_TAG}
                    docker push ${DOCKER_REPO}:latest
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {

                withCredentials([
                    file(
                        credentialsId: "${KUBE_CONFIG_ID}",
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        echo "===== Kubernetes Configuration ====="

                        echo "Kubernetes Contexts:"
                        kubectl config get-contexts

                        echo "Current Kubernetes Context:"
                        kubectl config current-context

                        echo "===== Kubernetes Cluster ====="
                        kubectl cluster-info

                        echo "===== Kubernetes Nodes ====="
                        kubectl get nodes
                    '''

                    sh '''
                        echo "===== Creating Zabbix Namespace ====="

                        kubectl get namespace ${K8S_NAMESPACE} \
                        || kubectl create namespace ${K8S_NAMESPACE}
                    '''
                }
            }
        }

        stage('Deploy PostgreSQL') {
            steps {

                withCredentials([
                    file(
                        credentialsId: "${KUBE_CONFIG_ID}",
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        echo "===== Deploying PostgreSQL ====="

                        kubectl apply \
                            -f k8s/postgres.yaml \
                            -n ${K8S_NAMESPACE}
                    '''
                }
            }
        }

        stage('Deploy Zabbix Server') {
            steps {

                withCredentials([
                    file(
                        credentialsId: "${KUBE_CONFIG_ID}",
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        echo "===== Deploying Zabbix Server ====="

                        kubectl apply \
                            -f k8s/zabbix-server.yaml \
                            -n ${K8S_NAMESPACE}
                    '''
                }
            }
        }

        stage('Deploy Zabbix Web') {
            steps {

                withCredentials([
                    file(
                        credentialsId: "${KUBE_CONFIG_ID}",
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        echo "===== Deploying Zabbix Web ====="

                        kubectl apply \
                            -f k8s/zabbix-web.yaml \
                            -n ${K8S_NAMESPACE}
                    '''
                }
            }
        }

        stage('Apply Services') {
            steps {

                withCredentials([
                    file(
                        credentialsId: "${KUBE_CONFIG_ID}",
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        echo "===== Applying Zabbix Services ====="

                        kubectl apply \
                            -f k8s/zabbix-service.yaml \
                            -n ${K8S_NAMESPACE}
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {

                withCredentials([
                    file(
                        credentialsId: "${KUBE_CONFIG_ID}",
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh '''
                        echo "=========================================="
                        echo "Zabbix Kubernetes Deployment Status"
                        echo "=========================================="

                        echo "===== Pods ====="
                        kubectl get pods -n ${K8S_NAMESPACE} -o wide

                        echo "===== Deployments ====="
                        kubectl get deployments -n ${K8S_NAMESPACE}

                        echo "===== Services ====="
                        kubectl get services -n ${K8S_NAMESPACE}
                    '''
                }
            }
        }
    }

    post {

        success {
            echo '=========================================='
            echo 'Zabbix deployment completed successfully'
            echo '=========================================='
        }

        failure {
            echo '=========================================='
            echo 'Zabbix deployment FAILED'
            echo '=========================================='
        }

        always {
            sh 'docker logout || true'
        }
    }
}
```
