pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(
            logRotator(
                numToKeepStr: '10',
                artifactNumToKeepStr: '5'
            )
        )
    }

    environment {
        SWR_REGISTRY   = 'swr.me-east-6001.partnergateway.dutech-cloud.ae'
        SWR_NAMESPACE  = 'myproject'
        IMAGE_NAME     = 'above-education'

        K8S_NAMESPACE  = 'education'
        DEPLOYMENT     = 'above-education'
        CONTAINER_NAME = 'above-education'

        FULL_IMAGE     = "${SWR_REGISTRY}/${SWR_NAMESPACE}/${IMAGE_NAME}:${BUILD_NUMBER}"
        LATEST_IMAGE   = "${SWR_REGISTRY}/${SWR_NAMESPACE}/${IMAGE_NAME}:latest"
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm

                sh '''
                    echo "Current Git commit:"
                    git rev-parse HEAD

                    echo "Checking required files..."

                    test -f Dockerfile || {
                        echo "ERROR: Dockerfile is missing"
                        exit 1
                    }

                    test -f index.html || {
                        echo "ERROR: index.html is missing"
                        exit 1
                    }

                    test -f img/slides/1.jpg || {
                        echo "ERROR: img/slides/1.jpg is missing"
                        exit 1
                    }

                    echo "Updated banner image details:"
                    ls -lh img/slides/1.jpg
                    sha256sum img/slides/1.jpg
                '''
            }
        }

        stage('Check Docker Access') {
            steps {
                sh '''
                    docker --version
                    docker info >/dev/null
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker images:"
                    echo "$FULL_IMAGE"
                    echo "$LATEST_IMAGE"

                    docker build \
                      --pull \
                      --no-cache \
                      -t "$FULL_IMAGE" \
                      -t "$LATEST_IMAGE" \
                      .
                '''
            }
        }

        stage('Verify Built Image') {
            steps {
                sh '''
                    echo "Verifying the banner inside the built image..."

                    CONTAINER_ID=$(docker create "$FULL_IMAGE")

                    docker cp \
                      "$CONTAINER_ID:/usr/share/nginx/html/img/slides/1.jpg" \
                      /tmp/jenkins-built-1.jpg

                    docker rm "$CONTAINER_ID"

                    echo "GitHub workspace image:"
                    sha256sum img/slides/1.jpg

                    echo "Image inside Docker container:"
                    sha256sum /tmp/jenkins-built-1.jpg

                    cmp img/slides/1.jpg /tmp/jenkins-built-1.jpg || {
                        echo "ERROR: Image inside Docker image does not match GitHub file"
                        rm -f /tmp/jenkins-built-1.jpg
                        exit 1
                    }

                    rm -f /tmp/jenkins-built-1.jpg

                    echo "Docker image verification successful"
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

                        if [ -z "$SWR_USER" ]; then
                            echo "ERROR: SWR username is empty"
                            exit 1
                        fi

                        if [ -z "$SWR_PASSWORD" ]; then
                            echo "ERROR: SWR password is empty"
                            exit 1
                        fi

                        echo "$SWR_PASSWORD" | docker login \
                          "$SWR_REGISTRY" \
                          --username "$SWR_USER" \
                          --password-stdin

                        echo "SWR login successful"
                    '''
                }
            }
        }

        stage('Push Image to SWR') {
            steps {
                sh '''
                    echo "Pushing numbered image:"
                    echo "$FULL_IMAGE"
                    docker push "$FULL_IMAGE"

                    echo "Updating latest image:"
                    echo "$LATEST_IMAGE"
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

                        echo "Checking CCE connectivity..."
                        kubectl cluster-info
                        kubectl get nodes

                        echo "Checking existing deployment..."
                        kubectl get deployment "$DEPLOYMENT" \
                          -n "$K8S_NAMESPACE"

                        echo "Updating deployment image to:"
                        echo "$FULL_IMAGE"

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

                        echo "Deployment image currently configured:"
                        kubectl get deployment "$DEPLOYMENT" \
                          -n "$K8S_NAMESPACE" \
                          -o jsonpath='{.spec.template.spec.containers[*].image}'

                        echo
                        echo "Deployment status:"
                        kubectl get deployment "$DEPLOYMENT" \
                          -n "$K8S_NAMESPACE" \
                          -o wide

                        echo
                        echo "Pods:"
                        kubectl get pods \
                          -n "$K8S_NAMESPACE" \
                          -o wide

                        echo
                        echo "Services:"
                        kubectl get service \
                          -n "$K8S_NAMESPACE"
                    '''
                }
            }
        }

        stage('Verify Image Inside Running Pod') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'cce-kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        POD_NAME=$(kubectl get pods \
                          -n "$K8S_NAMESPACE" \
                          -l app="$DEPLOYMENT" \
                          --field-selector=status.phase=Running \
                          -o jsonpath='{.items[0].metadata.name}')

                        if [ -z "$POD_NAME" ]; then
                            echo "No pod found with label app=$DEPLOYMENT"
                            echo "Available pod labels:"
                            kubectl get pods \
                              -n "$K8S_NAMESPACE" \
                              --show-labels
                            exit 1
                        fi

                        echo "Checking pod: $POD_NAME"

                        kubectl exec \
                          -n "$K8S_NAMESPACE" \
                          "$POD_NAME" \
                          -- ls -lh \
                          /usr/share/nginx/html/img/slides/1.jpg

                        echo "Checksum inside running pod:"
                        kubectl exec \
                          -n "$K8S_NAMESPACE" \
                          "$POD_NAME" \
                          -- sha256sum \
                          /usr/share/nginx/html/img/slides/1.jpg
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
            echo "Image pushed: ${FULL_IMAGE}"
            echo "Latest tag updated: ${LATEST_IMAGE}"
        }

        failure {
            echo 'Pipeline failed. Check the first failed stage in the console output.'
        }

        always {
            sh '''
                docker logout "$SWR_REGISTRY" || true

                docker image rm "$FULL_IMAGE" || true
                docker image rm "$LATEST_IMAGE" || true

                docker image prune -f || true
            '''
        }
    }
}
