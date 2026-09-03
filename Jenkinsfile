pipeline {
    agent any

    stages {

        stage('Build Backend Image') {
            steps {
                sh '''
                    docker build \
                        -t ecommerce-backend:${BUILD_NUMBER} \
                        ./src/server
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                    docker build \
                        -t ecommerce-frontend:${BUILD_NUMBER} \
                        ./src/client
                '''
            }
        }
    }
}