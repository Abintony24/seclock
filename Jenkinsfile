pipeline {
    agent any

    environment {
        AWS_REGION     = "us-east-1"
        AWS_ACCOUNT_ID = "504483085764"
        ECR_REPO       = "seclock"
        ECR_REGISTRY   = "504483085764.dkr.ecr.us-east-1.amazonaws.com"
        IMAGE          = "504483085764.dkr.ecr.us-east-1.amazonaws.com/seclock"
        EKS_CLUSTER    = "beginner-cluster"
    }

    options {
        skipDefaultCheckout(true)
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m pip install --break-system-packages -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    python3 -m compileall .
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('sonarqube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=seclock \
                            -Dsonar.projectName='Seclock' \
                            -Dsonar.sources=. \
                            -Dsonar.sourceEncoding=UTF-8
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE}:${BUILD_NUMBER} .
                    docker tag ${IMAGE}:${BUILD_NUMBER} ${IMAGE}:latest
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}

                        docker push ${IMAGE}:${BUILD_NUMBER}
                        docker push ${IMAGE}:latest
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        aws eks update-kubeconfig \
                            --region ${AWS_REGION} \
                            --name ${EKS_CLUSTER}

                        sed -i "s|image: .*|image: ${IMAGE}:${BUILD_NUMBER}|g" \
                            k8s/deployment.yaml

                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml

                        kubectl rollout status deployment/seclock
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
