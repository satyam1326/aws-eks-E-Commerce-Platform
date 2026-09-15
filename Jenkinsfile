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
    stage('ECR Login') {
        steps {
            sh '''
                aws ecr get-login-password --region ${AWS_REGION} | \
                docker login \
                    --username AWS \
                    --password-stdin ${ECR_REPOSITORY}
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

    stage('Configure EKS Access') {
        steps {
            sh '''
                echo "========================================"
                echo "Configuring kubectl for EKS"
                echo "========================================"

                aws eks update-kubeconfig \
                    --region ${AWS_REGION} \
                    --name dev-eks

                kubectl get nodes
            '''
        }
    }

    stage('Run Prisma Migrations') {
        steps {
            sh '''
                echo "========================================"
                echo "Running Prisma Database Migrations"
                echo "========================================"

                kubectl delete pod prisma-migrate --ignore-not-found

                cat <<EOF > prisma-migrate.yaml
apiVersion: v1
kind: Pod
metadata:
  name: prisma-migrate
spec:
  restartPolicy: Never
  containers:
    - name: prisma-migrate
      image: ${ECR_REPOSITORY}:backend-${BUILD_NUMBER}
      command:
        - npx
        - prisma
        - migrate
        - deploy
      envFrom:
        - secretRef:
            name: backend-secret
EOF

                kubectl apply -f prisma-migrate.yaml

                kubectl wait \
                    --for=jsonpath='{.status.phase}'=Succeeded \
                    pod/prisma-migrate \
                    --timeout=300s

                kubectl logs prisma-migrate

                kubectl delete pod prisma-migrate --ignore-not-found

                rm -f prisma-migrate.yaml
            '''
        }
    }

    stage('Deploy Application to EKS') {
        steps {
            sh '''
                echo "========================================"
                echo "Deploying Application to EKS"
                echo "========================================"

                kubectl apply -f kubernetes/backend/deployment.yaml
                kubectl apply -f kubernetes/backend/service.yaml

                kubectl apply -f kubernetes/frontend/frontend.yaml

                kubectl apply -f kubernetes/ingress/alb-ingress.yaml
            '''
        }
    }

    stage('Update EKS Images') {
        steps {
            sh '''
                echo "========================================"
                echo "Updating Application Images"
                echo "========================================"

                kubectl set image deployment/backend \
                backend=${ECR_REPOSITORY}:backend-${BUILD_NUMBER}

                kubectl set image deployment/frontend \
                frontend=${ECR_REPOSITORY}:frontend-${BUILD_NUMBER}
            '''
        }
    }

    stage('Verify EKS Rollout') {
        steps {
            sh '''
                echo "========================================"
                echo "Verifying Kubernetes Rollout"
                echo "========================================"

                kubectl rollout status deployment/backend \
                    --timeout=300s

                kubectl rollout status deployment/frontend \
                    --timeout=300s

                echo "========================================"
                echo "Current Application Pods"
                echo "========================================"

                kubectl get pods -o wide
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