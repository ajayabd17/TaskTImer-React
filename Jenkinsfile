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

        stage('Preflight / Kubeconfig check') {
            steps {
                script {
                    try {
                        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                            if (isUnix()) {
                                sh "echo 'Using kubeconfig:' ${KUBECONFIG_FILE} || true"
                                sh "ls -l ${KUBECONFIG_FILE} || true"
                                sh "KUBECONFIG=${KUBECONFIG_FILE} kubectl version --client || true"
                                sh "KUBECONFIG=${KUBECONFIG_FILE} kubectl cluster-info || true"
                            } else {
                                bat "echo Using kubeconfig: %KUBECONFIG_FILE%"
                                bat "type %KUBECONFIG_FILE%"
                                bat "set KUBECONFIG=%KUBECONFIG_FILE% && kubectl version --client || echo kubectl-not-found"
                                bat "set KUBECONFIG=%KUBECONFIG_FILE% && kubectl cluster-info || echo cluster-info-failed"
                            }
                        }
                    } catch (e) {
                        echo "WARNING: kubeconfig credential 'kubeconfig' not available or invalid. Please add it in Jenkins credentials (Secret file, id=kubeconfig)."
                        error("Missing or invalid kubeconfig credential: ${e}")
                    }
                }
            }
        }

        stage('Docker Build and Tag') {
            steps {
                script {
                    if (isUnix()) {
                        sh '''
                            docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                            docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                            if command -v minikube >/dev/null 2>&1; then
                              echo "minikube detected — loading image into minikube"
                              minikube image load ${DOCKER_IMAGE}:${BUILD_NUMBER} || true
                            fi
                        '''
                    } else {
                        bat """
                            docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                            docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                        """
                        bat 'where minikube || echo minikube-not-found'
                        bat "minikube image load ${DOCKER_IMAGE}:${BUILD_NUMBER} || echo minikube-load-failed"
                    }
                }
            }
        }

        stage('Get Active Deployment') {
            steps {
                script {
                    try {
                        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                            def activeColor = isUnix() ?
                                sh(script: "KUBECONFIG=${KUBECONFIG_FILE} kubectl get service timer-app-service -n ${KUBE_NAMESPACE} -o jsonpath='{.spec.selector.track}'", returnStdout: true).trim() :
                                bat(script: "set KUBECONFIG=%KUBECONFIG_FILE% && kubectl get service timer-app-service -n ${KUBE_NAMESPACE} -o jsonpath='{.spec.selector.track}'", returnStdout: true).trim()

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
                        if (isUnix()) {
                            sh """
                                KUBECONFIG=${KUBECONFIG_FILE} kubectl set image deployment/timer-app-${env.TARGET_COLOR} timer-app=${DOCKER_IMAGE}:${BUILD_NUMBER} -n ${KUBE_NAMESPACE}
                                KUBECONFIG=${KUBECONFIG_FILE} kubectl rollout status deployment/timer-app-${env.TARGET_COLOR} -n ${KUBE_NAMESPACE} --timeout=300s
                            """
                        } else {
                            bat """
                                set KUBECONFIG=%KUBECONFIG_FILE% && kubectl set image deployment/timer-app-${env.TARGET_COLOR} timer-app=${DOCKER_IMAGE}:${BUILD_NUMBER} -n ${KUBE_NAMESPACE}
                                set KUBECONFIG=%KUBECONFIG_FILE% && kubectl rollout status deployment/timer-app-${env.TARGET_COLOR} -n ${KUBE_NAMESPACE} --timeout=300s
                            """
                        }
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        if (isUnix()) {
                            sh "KUBECONFIG=${KUBECONFIG_FILE} kubectl get pods -n ${KUBE_NAMESPACE} -l app=timer-app,track=${env.TARGET_COLOR} || true"
                        } else {
                            bat "set KUBECONFIG=%KUBECONFIG_FILE% && kubectl get pods -n ${KUBE_NAMESPACE} -l app=timer-app,track=${env.TARGET_COLOR} || echo no-pods"
                        }
                    }
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        if (isUnix()) {
                            sh "KUBECONFIG=${KUBECONFIG_FILE} kubectl patch service timer-app-service -n ${KUBE_NAMESPACE} --type=merge -p '{\"spec\":{\"selector\":{\"app\":\"timer-app\",\"track\":\"${env.TARGET_COLOR}\"}}}'"
                        } else {
                            bat "set KUBECONFIG=%KUBECONFIG_FILE% && kubectl patch service timer-app-service -n ${KUBE_NAMESPACE} --type=merge -p \"{\\\"spec\\\":{\\\"selector\\\":{\\\"app\\\":\\\"timer-app\\\",\\\"track\\\":\\\"${env.TARGET_COLOR}\\\"}}}\""
                        }
                    }
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        if (isUnix()) {
                            sh """\
KUBECONFIG=${KUBECONFIG_FILE} POD=$(kubectl get pods -n ${KUBE_NAMESPACE} -l app=timer-app,track=${env.TARGET_COLOR} -o jsonpath='{.items[0].metadata.name}') || true
if [ -n "$POD" ]; then
  KUBECONFIG=${KUBECONFIG_FILE} kubectl exec -n ${KUBE_NAMESPACE} $POD -- curl -f http://localhost:80 --connect-timeout 10 || true
fi
"""
                        } else {
                            bat """
set KUBECONFIG=%KUBECONFIG_FILE% && for /f \"tokens=*\" %%p in ('kubectl get pods -n ${KUBE_NAMESPACE} -l app=timer-app,track=${env.TARGET_COLOR} -o jsonpath=\"{.items[0].metadata.name}\"') do set POD=%%p
set KUBECONFIG=%KUBECONFIG_FILE% && if defined POD kubectl exec -n ${KUBE_NAMESPACE} %POD% -- curl -f http://localhost:80 --connect-timeout 10 || echo healthcheck-failed
"""
                        }
                    }
                }
            }
        }

        stage('Cleanup Old Deployment') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        if (isUnix()) {
                            sh "KUBECONFIG=${KUBECONFIG_FILE} kubectl scale deployment timer-app-${env.ACTIVE_COLOR} -n ${KUBE_NAMESPACE} --replicas=0 || true"
                        } else {
                            bat "set KUBECONFIG=%KUBECONFIG_FILE% && kubectl scale deployment timer-app-${env.ACTIVE_COLOR} -n ${KUBE_NAMESPACE} --replicas=0 || echo scale-failed"
                        }
                    }
                    sh 'docker image prune -f || true'
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
                    if (isUnix()) {
                        sh "KUBECONFIG=${KUBECONFIG_FILE} kubectl patch service timer-app-service -n ${KUBE_NAMESPACE} --type=merge -p '{\\"spec\\":{\\"selector\\":{\\"app\\":\\"timer-app\\\",\\"track\\":\\"${env.ACTIVE_COLOR}\\\"}}}' || true"
                        sh "KUBECONFIG=${KUBECONFIG_FILE} kubectl rollout undo deployment/timer-app-${env.TARGET_COLOR} -n ${KUBE_NAMESPACE} || true"
                    } else {
                        bat "set KUBECONFIG=%KUBECONFIG_FILE% && kubectl patch service timer-app-service -n ${KUBE_NAMESPACE} --type=merge -p \"{\\\"spec\\\":{\\\"selector\\\":{\\\"app\\\":\\\"timer-app\\\",\\\"track\\\":\\\"${env.ACTIVE_COLOR}\\\"}}}\" || echo patch-failed"
                        bat "set KUBECONFIG=%KUBECONFIG_FILE% && kubectl rollout undo deployment/timer-app-${env.TARGET_COLOR} -n ${KUBE_NAMESPACE} || echo undo-failed"
                    }
                }
            }
        }
        always {
            cleanWs()
        }
    }
}
