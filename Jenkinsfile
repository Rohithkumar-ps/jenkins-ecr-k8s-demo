pipeline {
    agent any

    environment {
        AWS_REGION   = 'ap-south-1'
        AWS_ACCOUNT  = '590385682114'
        ECR_REPO     = 'cicd-demo-app'
        ECR_REGISTRY = '590385682114.dkr.ecr.ap-south-1.amazonaws.com'
        IMAGE_URI    = "590385682114.dkr.ecr.ap-south-1.amazonaws.com/cicd-demo-app:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarCloud Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarCloud') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        echo "Quality Gate status: ${qg.status}"
                        if (qg.status != 'OK' && qg.status != 'NONE') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t "$IMAGE_URI" .'
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    set -e

                    aws ecr get-login-password --region "$AWS_REGION" | \
                        docker login --username AWS --password-stdin "$ECR_REGISTRY"

                    docker push "$IMAGE_URI"
                '''
            }
        }

        stage('Configure Kubernetes ECR Access') {
            steps {
                sh '''
                    set -e

                    kubectl create secret docker-registry ecr-registry \
                        --docker-server="$ECR_REGISTRY" \
                        --docker-username=AWS \
                        --docker-password="$(aws ecr get-login-password --region "$AWS_REGION")" \
                        --dry-run=client -o yaml | kubectl apply -f -
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    set -e

                    sed "s|IMAGE_PLACEHOLDER|$IMAGE_URI|g" \
                        k8s/deployment.yaml | kubectl apply -f -

                    kubectl rollout status deployment/cicd-demo-app \
                        --timeout=180s

                    kubectl get deployment cicd-demo-app
                    kubectl get pods -o wide
                    kubectl get service cicd-demo-app
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment successful: ${IMAGE_URI}"
        }

        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
