pipeline {

    agent any

    environment {
        NAMESPACE = 'zabbix'
        K8S_CONTEXT = 'docker-desktop'

        ZABBIX_SERVER_IMAGE = 'zabbix/zabbix-server-pgsql:7.0-ubuntu-latest'
        ZABBIX_WEB_IMAGE    = 'zabbix/zabbix-web-nginx-pgsql:7.0-ubuntu-latest'
        POSTGRES_IMAGE      = 'postgres:16-alpine'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code from GitHub...'

                checkout scm
            }
        }

        stage('Verify Files') {
            steps {
                bat '''
                    echo ===== Repository Files =====
                    dir

                    echo ===== Kubernetes Files =====
                    dir k8s
                '''
            }
        }

        stage('Kubernetes Context') {
            steps {
                bat '''
                    echo ===== Kubernetes Context =====

                    kubectl config get-contexts

                    kubectl config use-context %K8S_CONTEXT%

                    kubectl cluster-info
                '''
            }
        }

        stage('Create Namespace') {
            steps {
                bat '''
                    kubectl apply -f k8s/namespace.yaml
                '''
            }
        }

        stage('Deploy PostgreSQL') {
            steps {
                bat '''
                    kubectl apply -f k8s/postgres.yaml -n %NAMESPACE%

                    echo Waiting for PostgreSQL...

                    kubectl rollout status deployment/zabbix-postgres ^
                        -n %NAMESPACE% ^
                        --timeout=180s
                '''
            }
        }

        stage('Deploy Zabbix Server') {
            steps {
                bat '''
                    kubectl apply -f k8s/zabbix-server.yaml -n %NAMESPACE%

                    echo Waiting for Zabbix Server...

                    kubectl rollout status deployment/zabbix-server ^
                        -n %NAMESPACE% ^
                        --timeout=300s
                '''
            }
        }

        stage('Deploy Zabbix Web') {
            steps {
                bat '''
                    kubectl apply -f k8s/zabbix-web.yaml -n %NAMESPACE%

                    echo Waiting for Zabbix Web...

                    kubectl rollout status deployment/zabbix-web ^
                        -n %NAMESPACE% ^
                        --timeout=300s
                '''
            }
        }

        stage('Apply Services') {
            steps {
                bat '''
                    kubectl apply -f k8s/zabbix-service.yaml -n %NAMESPACE%
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                bat '''
                    echo ==============================
                    echo Zabbix Pods
                    echo ==============================

                    kubectl get pods -n %NAMESPACE% -o wide

                    echo ==============================
                    echo Zabbix Services
                    echo ==============================

                    kubectl get svc -n %NAMESPACE%

                    echo ==============================
                    echo Deployments
                    echo ==============================

                    kubectl get deployments -n %NAMESPACE%
                '''
            }
        }
    }

    post {

        success {
            echo '======================================'
            echo 'Zabbix deployment completed successfully'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'Zabbix deployment FAILED'
            echo '======================================'

            bat '''
                kubectl get pods -n %NAMESPACE%
                kubectl get events -n %NAMESPACE% --sort-by=.lastTimestamp
            '''
        }
    }
}