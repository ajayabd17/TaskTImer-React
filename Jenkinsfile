pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/ajayabd17/TaskTImer-React.git'
            }
        }

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

        stage('Docker Build (optional)') {
            steps {
                bat 'docker build -t timer-app .'
            }
        }
    }

    post {
        success {
            echo '✅ Build successful!'
        }
        failure {
            echo '❌ Build failed!'
        }
        always {
            cleanWs()
        }
    }
}
