pipeline {
    agent any

    environment {
        IMAGE_NAME = 'swr.me-east-6001.partnergateway.dutech-cloud.ae/myproject/sr-nginx'
        IMAGE_TAG  = "${BUILD_NUMBER}"
        NAMESPACE  = 'default'
        DEPLOYMENT = 'sr-nginx'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build \
                    -t ${IMAGE_NAME}:${IMAGE_TAG} \
                    -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Login to HCS SWR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'hcs-swr-credentials',
                        usernameVariable: 'SWR_USER',
                        passwordVariable: 'SWR_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "${SWR_PASSWORD}" | \
                        docker login \
                        swr.me-east-6001.partnergateway.dutech-cloud.ae \
                        -u "${SWR_USER}" \
                        --password-stdin
                    '''
                }
            }
        }

        stage('Push Image to SWR') {
            steps {
                sh '''
                    echo "Pushing Docker images to HCS SWR..."

                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Deploy to CCE') {
            steps {
                sh '''
                    echo "Updating CCE deployment..."

                    kubectl set image \
                    deployment/${DEPLOYMENT} \
                    sr-nginx=${IMAGE_NAME}:${IMAGE_TAG} \
                    -n ${NAMESPACE}

                    kubectl rollout status \
                    deployment/${DEPLOYMENT} \
                    -n ${NAMESPACE} \
                    --timeout=300s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Checking deployment..."
                    kubectl get deployment ${DEPLOYMENT} \
                    -n ${NAMESPACE}

                    echo "Checking pods..."
                    kubectl get pods \
                    -n ${NAMESPACE} \
                    -o wide

                    echo "Checking services..."
                    kubectl get svc \
                    -n ${NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo 'Website image built, pushed to HCS SWR, and deployed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }

        always {
            sh '''
                docker logout \
                swr.me-east-6001.partnergateway.dutech-cloud.ae || true

                docker image rm \
                ${IMAGE_NAME}:${IMAGE_TAG} || true
            '''
        }
    }
}
