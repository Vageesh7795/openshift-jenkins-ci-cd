pipeline {
    agent any

    environment {
        PROJECT  = 'vageeshtk-dev'
        APP_NAME = 'my-application'

        OC_SERVER = 'https://172.30.0.1:443'

        REGISTRY = 'image-registry.openshift-image-registry.svc:5000'
        IMAGE    = "${REGISTRY}/${PROJECT}/${APP_NAME}"
        TAG      = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out application source from GitHub...'
                checkout scm
            }
        }

        stage('Login to OpenShift') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'openshift-token',
                        variable: 'OC_TOKEN'
                    )
                ]) {
                    sh '''
                        oc login \
                          --server="${OC_SERVER}" \
                          --token="${OC_TOKEN}" \
                          --insecure-skip-tls-verify=true

                        oc project "${PROJECT}"

                        echo "Logged in user:"
                        oc whoami

                        echo "Current project:"
                        oc project
                    '''
                }
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    echo "Building Docker image..."

                    podman build \
                      -t "${IMAGE}:${TAG}" \
                      -t "${IMAGE}:latest" \
                      .
                '''
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    echo "Logging into OpenShift registry..."

                    oc registry login

                    echo "Pushing image..."

                    podman push "${IMAGE}:${TAG}"
                    podman push "${IMAGE}:latest"
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    echo "Deploying application..."

                    oc apply -f k8s/deployment.yaml
                    oc apply -f k8s/service.yaml
                    oc apply -f k8s/route.yaml

                    echo "Updating deployment image..."

                    oc set image deployment/${APP_NAME} \
                      ${APP_NAME}=${IMAGE}:${TAG}

                    echo "Waiting for rollout..."

                    oc rollout status deployment/${APP_NAME}
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    echo "===== PODS ====="
                    oc get pods -o wide

                    echo "===== SERVICE ====="
                    oc get svc ${APP_NAME}

                    echo "===== ROUTE ====="
                    oc get route ${APP_NAME}
                '''
            }
        }
    }

    post {
        success {
            echo '===================================='
            echo 'Application deployed successfully!'
            echo '===================================='
        }

        failure {
            echo '===================================='
            echo 'Deployment failed!'
            echo 'Check the Jenkins console output.'
            echo '===================================='
        }
    }
}
