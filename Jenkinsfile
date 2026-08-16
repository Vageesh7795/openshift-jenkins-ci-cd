pipeline {
    agent any

    environment {
        PROJECT = 'vageeshtk-dev'
        APP_NAME = 'my-application'

        OC_SERVER = 'https://api.YOUR-CLUSTER:6443'
        OC_TOKEN = 'sha256~YOUR_SERVICE_ACCOUNT_TOKEN'

        IMAGE = "image-registry.openshift-image-registry.svc:5000/${PROJECT}/${APP_NAME}"
        TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out application source...'
                checkout scm
            }
        }

        stage('Login to OpenShift') {
            steps {
                sh '''
                    echo "Logging into OpenShift..."

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

        stage('Create BuildConfig') {
            steps {
                sh '''
                    echo "Checking BuildConfig..."

                    if oc get buildconfig "${APP_NAME}" >/dev/null 2>&1
                    then
                        echo "BuildConfig already exists."
                    else
                        echo "Creating BuildConfig..."

                        oc new-build \
                          --binary \
                          --name="${APP_NAME}" \
                          --strategy=docker
                    fi

                    echo "BuildConfig:"
                    oc get buildconfig "${APP_NAME}"
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    echo "Starting OpenShift build..."

                    oc start-build "${APP_NAME}" \
                      --from-dir=. \
                      --follow

                    echo "Build completed."

                    oc get builds
                '''
            }
        }

        stage('Tag Image') {
            steps {
                sh '''
                    echo "Creating build-number image tag..."

                    oc tag \
                      "${APP_NAME}:latest" \
                      "${APP_NAME}:${TAG}"

                    echo "ImageStream tags:"
                    oc get imagestreamtag "${APP_NAME}:latest"
                    oc get imagestreamtag "${APP_NAME}:${TAG}"
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    echo "Applying Kubernetes/OpenShift resources..."

                    oc apply -f k8s/deployment.yaml
                    oc apply -f k8s/service.yaml
                    oc apply -f k8s/route.yaml

                    echo "Updating deployment image..."

                    oc set image deployment/"${APP_NAME}" \
                      "${APP_NAME}"="${IMAGE}:${TAG}"

                    echo "Waiting for rollout..."

                    oc rollout status deployment/"${APP_NAME}" \
                      --timeout=5m
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    echo "======================================"
                    echo "PODS"
                    echo "======================================"

                    oc get pods -o wide

                    echo "======================================"
                    echo "DEPLOYMENT"
                    echo "======================================"

                    oc get deployment "${APP_NAME}"

                    echo "======================================"
                    echo "SERVICE"
                    echo "======================================"

                    oc get service "${APP_NAME}"

                    echo "======================================"
                    echo "ROUTE"
                    echo "======================================"

                    oc get route "${APP_NAME}"

                    echo "======================================"
                    echo "APPLICATION URL"
                    echo "======================================"

                    oc get route "${APP_NAME}" \
                      -o jsonpath='{.spec.host}'

                    echo
                '''
            }
        }
    }

    post {
        success {
            echo '''
========================================
 APPLICATION DEPLOYED SUCCESSFULLY
========================================
'''
        }

        failure {
            echo '''
========================================
 DEPLOYMENT FAILED
========================================
Check the Jenkins console output.
'''
        }
    }
}
