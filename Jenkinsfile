pipeline {
    agent any

    environment {
        PROJECT = 'vageeshtk-dev'
        APP_NAME = 'my-application'

        OC_SERVER = 'https://172.30.0.1:443'
        OC_TOKEN = 'eyJhbGciOiJSUzI1NiIsImtpZCI6IkZYZUc1d1FpY1BJV0lfNlNjRUhRM1hHOHE0NWd0X2cxY1o0TXUwQV9lMWcifQ.eyJhdWQiOlsiaHR0cHM6Ly9vaWRjLm9wMS5vcGVuc2hpZnRhcHBzLmNvbS8yZmVnZnU5bHNnNDdpNzF0a2ozcGJsbjR0c29yZGVjNyJdLCJleHAiOjE3ODY4OTk0NDEsImlhdCI6MTc4Njg5NTg0MSwiaXNzIjoiaHR0cHM6Ly9vaWRjLm9wMS5vcGVuc2hpZnRhcHBzLmNvbS8yZmVnZnU5bHNnNDdpNzF0a2ozcGJsbjR0c29yZGVjNyIsImp0aSI6IjczZWViMmMxLWE0ZjgtNGRjZS05YzU2LTUyZTc4ZDBlYjVmZiIsImt1YmVybmV0ZXMuaW8iOnsibmFtZXNwYWNlIjoidmFnZWVzaHRrLWRldiIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJqZW5raW5zLWRlcGxveWVyIiwidWlkIjoiOGJhY2QwOTktODNlZS00ZWJhLWE0NGMtYmM1M2UyOTA0MWY2In19LCJuYmYiOjE3ODY4OTU4NDEsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDp2YWdlZXNodGstZGV2OmplbmtpbnMtZGVwbG95ZXIifQ.UTJXXMCNuNwKDxN0owdxJdjqpaf_6ZxKno9Q0LkBHmrDuL-y8c7XZsS5RXNniPlGRo6PpKdEyQpp1C9MKQo-9jMwr3R2jlA13oNQZzIpS5hVIM-keC8BEOm7Bmcqet9L8GcTVnLiPdfCjAdTzy2korhwrKYBIbLcHvhDOOfSs_bnubVc3-BrB2l0LeCZq0Mp4uitenoXo16vMyGReJB0HpKhL1-5-cGOFF72z0h4ycX-pvTKILO6KbN5ZOWo1mbpvKmWLpJt6azjUJdrGvEeE6iTHjq9gIQUinJGtnMX3nOZKO-_YoInCifXbyzd6yiqXRvJgqxnOUvdKHZjhzJj0DYRC0zAMX6GmcvZObAAMFa42rJt88Mu_90hJpS2efiGXz2MxQpf70IpqBYGiHbq-QEnIicJLTnsK8Ym2DNDCMnGIxBcPG9MlanOI4WooZBG3tl2ogtiizJCJC9VPnsOzzY2eiC5pmbKL1WRsmy6Nbe9889Mdp6P9t7xnbstFE3UWPe0WB2fCgE4S0ARHWVEl0qhi77MmB6hbUlQ_7gP5QbQIhPP1iTLJX5Blc1z1udLQkWuhq_SfNlWwpBbHriDnRWN0iNLy-4bUtxwOJBxkgf3Hv2qKxrbEkI4mPIcjl0Nd--Hb_mwHH9WbNu_RbKKeY3IE3f_Ol_NXH9QVKJACZY'

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
                      --timeout=15m
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
