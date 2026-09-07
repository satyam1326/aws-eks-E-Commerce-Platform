pipeline {
agent any

environment {
    AWS_REGION = 'ap-south-1'
    ECR_REPOSITORY = '758209208592.dkr.ecr.ap-south-1.amazonaws.com/aws-eks-platform'
}

stages {

    stage('SonarQube Analysis') {
        steps {
            script {
                def scannerHome = tool 'sonar-scanner'

                withSonarQubeEnv('SonarQube') {
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=aws-ecommerce-platform -Dsonar.projectName='AWS E-Commerce Platform'"
                }
            }
        }
    }

    stage('Gitleaks Secrets Scan') {
        steps {
            sh '''
                echo "========================================"
                echo "Running Gitleaks Secrets Scan"
                echo "========================================"

                gitleaks dir \
                    . \
                    --config .gitleaks.toml \
                    --no-banner \
                    --exit-code 0
            '''
        }
    }

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

    stage('Trivy Image Scan') {
        steps {
            sh '''
                echo "========================================"
                echo "Scanning Backend Docker Image"
                echo "========================================"

                trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    ecommerce-backend:${BUILD_NUMBER}

                echo "========================================"
                echo "Scanning Frontend Docker Image"
                echo "========================================"

                trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 0 \
                    ecommerce-frontend:${BUILD_NUMBER}
            '''
        }
    }

    stage('Tag Images') {
        steps {
            sh '''
                docker tag \
                    ecommerce-backend:${BUILD_NUMBER} \
                    ${ECR_REPOSITORY}:backend-${BUILD_NUMBER}

                docker tag \
                    ecommerce-frontend:${BUILD_NUMBER} \
                    ${ECR_REPOSITORY}:frontend-${BUILD_NUMBER}
            '''
        }
    }

    stage('Push Images to ECR') {
        steps {
            sh '''
                docker push ${ECR_REPOSITORY}:backend-${BUILD_NUMBER}
                docker push ${ECR_REPOSITORY}:frontend-${BUILD_NUMBER}
            '''
        }
    }

    stage('Cleanup Docker Images') {
        steps {
            sh '''
                docker image prune -f
            '''
        }
    }
}

}