pipeline {
    agent any

    environment {
        SWR_REGISTRY   = 'swr.me-east-6001.partnergateway.dutech-cloud.ae'
        SWR_NAMESPACE  = 'myproject'
        IMAGE_NAME     = 'above-education'

        K8S_NAMESPACE  = 'education'
        DEPLOYMENT     = 'above-education'
        CONTAINER_NAME = 'above-education'

        IMAGE_TAG      = "${BUILD_NUMBER}"
        FULL_IMAGE     = "${SWR_REGISTRY}/${SWR_NAMESPACE}/${IMAGE_NAME}:${BUILD_NUMBER}"
        LATEST_IMAGE   = "${SWR_REGISTRY}/${SWR_NAMESPACE}/${IMAGE_NAME}:latest"
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
                    echo "Building $FULL_IMAGE"

                    docker build \
                      -t "$FULL_IMAGE" \
                      -t "$LATEST_IMAGE" \
                      .
                '''
            }
        }

        stage('Login to HCS SWR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'swr-credentials',
                        usernameVariable: 'SWR_USER',
                        passwordVariable: 'SWR_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x

                        test -n "$SWR_USER" || {
                            echo "SWR username is empty"
                            exit 1
                        }

                        test -n "$SWR_PASSWORD" || {
                            echo "SWR password is empty"
                            exit 1
                        }

                        echo "$SWR_PASSWORD" | docker login \
                          "$SWR_REGISTRY" \
                          --username "$SWR_USER" \
                          --password-stdin
                    '''
                }
            }
        }

        stage('Push Image to SWR') {
            steps {
                sh '''
                    docker push "$FULL_IMAGE"
                    docker push "$LATEST_IMAGE"
                '''
            }
        }

        stage('Deploy to CCE') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'cce-kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl get deployment "$DEPLOYMENT" \
                          -n "$K8S_NAMESPACE"

                        kubectl set image \
                          deployment/"$DEPLOYMENT" \
                          "$CONTAINER_NAME"="$FULL_IMAGE" \
                          -n "$K8S_NAMESPACE"

                        kubectl rollout status \
                          deployment/"$DEPLOYMENT" \
                          -n "$K8S_NAMESPACE" \
                          --timeout=300s
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'cce-kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl get pods -n "$K8S_NAMESPACE" -o wide
                        kubectl get service -n "$K8S_NAMESPACE"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
            echo "Image pushed: ${FULL_IMAGE}"
        }

        failure {
            echo "Pipeline failed. Check the first failed stage."
        }

        always {
            sh '''
                docker logout "$SWR_REGISTRY" || true
                docker image rm "$FULL_IMAGE" || true
                docker image rm "$LATEST_IMAGE" || true
            '''
        }
    }
}
