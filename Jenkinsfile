pipeline {

    agent any

    environment {

        // Docker Hub
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_REPO = 'dockercctns/zabbix'
	DOCKER_CREDS_ID = 'dockerhub-creds'
        IMAGE_TAG = "${BUILD_NUMBER}"

        // Kubernetes
        K8S_CONTEXT = 'docker-desktop'
        K8S_NAMESPACE = 'zabbix'
        KUBE_CONFIG_ID = 'k8s-kubeconfig'
	// TARGET_NS = "${env.BRANCH_NAME=='main' ? 'production'}"
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
                    echo ===== Repository =====
                    dir

                    echo ===== Kubernetes Files =====
                    dir k8s
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo ===== Building Docker Image =====

                    docker build ^
                        -t %DOCKER_REPO%:%IMAGE_TAG% ^
                        -t %DOCKER_REPO%:latest ^
                        -f docker/Dockerfile .
                '''
            }
        }

        stage('Docker Login') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'dockercctns',
                        passwordVariable: 'docker@123'
                    )
                ]) {

                    sh '''
                        echo ===== Docker Hub Login =====

                        docker login ^
                            -u %DOCKER_USERNAME% ^
                            -p %DOCKER_PASSWORD%
                    '''
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {

                sh '''
                    echo ===== Pushing Image =====

                    docker push %DOCKER_REPO%:%IMAGE_TAG%

                    docker push %DOCKER_REPO%:latest
                '''
            }
        }

        stage('Kubernetes Context') {
            steps {

                sh '''
                    echo ===== Kubernetes Context =====

                    kubectl config use-context %K8S_CONTEXT%

                    kubectl cluster-info

                    kubectl get nodes
                '''
            }
        }

        stage('Create Namespace') {
            steps {

                sh '''
                    kubectl apply -f k8s/namespace.yaml
                '''
            }
        }

        stage('Deploy PostgreSQL') {
            steps {

                sh '''
                    kubectl apply -f k8s/postgres.yaml ^
                        -n %K8S_NAMESPACE%

                    kubectl rollout status deployment/zabbix-postgres ^
                        -n %K8S_NAMESPACE% ^
                        --timeout=300s
                '''
            }
        }

        stage('Deploy Zabbix Server') {
            steps {

                sh '''
                    kubectl apply -f k8s/zabbix-server.yaml ^
                        -n %K8S_NAMESPACE%

                    kubectl rollout status deployment/zabbix-server ^
                        -n %K8S_NAMESPACE% ^
                        --timeout=300s
                '''
            }
        }

        stage('Deploy Zabbix Web') {
            steps {

                sh '''
                    kubectl apply -f k8s/zabbix-web.yaml ^
                        -n %K8S_NAMESPACE%

                    kubectl rollout status deployment/zabbix-web ^
                        -n %K8S_NAMESPACE% ^
                        --timeout=300s
                '''
            }
        }

        stage('Apply Services') {
            steps {

                sh '''
                    kubectl apply -f k8s/zabbix-service.yaml ^
                        -n %K8S_NAMESPACE%
                '''
            }
        }

        stage('Verify Deployment') {
            steps {

                sh '''
                    echo ==============================
                    echo PODS
                    echo ==============================

                    kubectl get pods ^
                        -n %K8S_NAMESPACE% ^
                        -o wide

                    echo ==============================
                    echo SERVICES
                    echo ==============================

                    kubectl get svc ^
                        -n %K8S_NAMESPACE%

                    echo ==============================
                    echo DEPLOYMENTS
                    echo ==============================

                    kubectl get deployments ^
                        -n %K8S_NAMESPACE%
                '''
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

            sh '''
                kubectl get pods -n %K8S_NAMESPACE%
                kubectl get events -n %K8S_NAMESPACE%
            '''
        }

        always {
            sh '''
                docker logout
            '''
        }
    }
}