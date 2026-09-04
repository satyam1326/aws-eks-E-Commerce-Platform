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

    stage('OWASP Dependency-Check') {
        steps {
            script {
                def dependencyCheckHome = tool 'dependency-check'

                withCredentials([
                    string(
                        credentialsId: 'nvd-api-key',
                        variable: 'NVD_API_KEY'
                    )
                ]) {
                    sh """
                        mkdir -p dependency-check-report

                        ${dependencyCheckHome}/bin/dependency-check.sh \
                            --project "AWS E-Commerce Platform" \
                            --scan ./src \
                            --format HTML \
                            --format JSON \
                            --out dependency-check-report \
                            --nvdApiKey "\${NVD_API_KEY}" \
                            --failOnCVSS 7
                    """
                }
            }
        }

        post {
            always {
                archiveArtifacts artifacts: 'dependency-check-report/*', allowEmptyArchive: true
            }
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