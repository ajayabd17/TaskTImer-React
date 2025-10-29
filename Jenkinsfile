pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'timer-app'
        KUBE_NAMESPACE = 'timer-app'
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        TARGET_COLOR = 'green'
        ACTIVE_COLOR = 'blue'
    }

    stages {
        stage('Install Dependencies') {
            steps {
                bat 'npm install'
            }
        }

        stage('Build') {
            steps {
                bat 'npm run build'
            }
        }

        stage('Docker Build and Tag') {
            steps {
                script {
                    bat """
                        docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                        docker build -t ${DOCKER_IMAGE}:latest .
                    """
                }
            }
        }

        stage('Get Active Deployment') {
            steps {
                script {
                    try {
                        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                            def activeColor = bat(script: "set KUBECONFIG=%KUBECONFIG_FILE% && kubectl get service timer-app-service -n ${KUBE_NAMESPACE} -o jsonpath='{.spec.selector.track}'", returnStdout: true).trim()

                            if (activeColor == 'blue') {
                                env.TARGET_COLOR = 'green'
                                env.ACTIVE_COLOR = 'blue'
                            } else {
                                env.TARGET_COLOR = 'blue'
                                env.ACTIVE_COLOR = 'green'
                            }

                            echo "Active Color: ${env.ACTIVE_COLOR}, Target Color: ${env.TARGET_COLOR}"
                        }
                    } catch (Exception e) {
                        echo "No active deployment found, defaulting blue->green"
                    }
                }
            }
        }

        stage('Deploy to Target Environment') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        bat """
                            set KUBECONFIG=%KUBECONFIG_FILE% && \
                            kubectl set image deployment/timer-app-${env.TARGET_COLOR} timer-app=${DOCKER_IMAGE}:${BUILD_NUMBER} -n ${KUBE_NAMESPACE} && \
                            kubectl rollout status deployment/timer-app-${env.TARGET_COLOR} -n ${KUBE_NAMESPACE} --timeout=300s
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        bat """
                            set KUBECONFIG=%KUBECONFIG_FILE% && kubectl get pods -n ${KUBE_NAMESPACE} -l app=timer-app,track=${env.TARGET_COLOR}
                        """
                    }
                    bat 'timeout /t 30 /nobreak'
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        bat """
                            set KUBECONFIG=%KUBECONFIG_FILE% && kubectl patch service timer-app-service -n ${KUBE_NAMESPACE} --type=merge -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"timer-app\\",\\"track\\":\\"${env.TARGET_COLOR}\\"}}}}"
                        """
                    }
                    bat 'timeout /t 10 /nobreak'
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        bat """
                            set KUBECONFIG=%KUBECONFIG_FILE% && kubectl get pods -n ${KUBE_NAMESPACE} -l app=timer-app,track=${env.TARGET_COLOR} -o jsonpath='{.items[0].metadata.name}' > pod_name.txt
                            set /p POD_NAME=<pod_name.txt
                            set KUBECONFIG=%KUBECONFIG_FILE% && kubectl exec -n ${KUBE_NAMESPACE} %POD_NAME% -- curl -f http://localhost:80 --connect-timeout 10
                        """
                    }
                }
            }
        }

        stage('Cleanup Old Deployment') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        bat """
                            set KUBECONFIG=%KUBECONFIG_FILE% && kubectl scale deployment timer-app-${env.ACTIVE_COLOR} -n ${KUBE_NAMESPACE} --replicas=0
                        """
                    }
                    bat 'docker image prune -f'
                }
            }
        }
    }

    post {
        success {
            echo "✅ Blue-Green deployment successful! New version running on ${env.TARGET_COLOR} environment"
        }
        failure {
            script {
                echo '❌ Deployment failed! Rolling back...'
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    bat """
                        set KUBECONFIG=%KUBECONFIG_FILE% && kubectl patch service timer-app-service -n ${KUBE_NAMESPACE} --type=merge -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"timer-app\\",\\"track\\":\\"${env.ACTIVE_COLOR}\\"}}}}" && \
                        set KUBECONFIG=%KUBECONFIG_FILE% && kubectl rollout undo deployment/timer-app-${env.TARGET_COLOR} -n ${KUBE_NAMESPACE}
                    """
                }
            }
        }
        always {
            cleanWs()
        }
    }
}
                     }
