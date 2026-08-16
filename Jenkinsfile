pipeline {
    agent any

    environment {
        PROJECT = 'vageeshtk-dev'
        APP_NAME = 'my-application'

        OC_SERVER = 'https://172.30.0.1:443'
        OC_TOKEN = 'eyJhbGciOiJSUzI1NiIsImtpZCI6IkZYZUc1d1FpY1BJV0lfNlNjRUhRM1hHOHE0NWd0X2cxY1o0TXUwQV9lMWcifQ.eyJhdWQiOlsiaHR0cHM6Ly9vaWRjLm9wMS5vcGVuc2hpZnRhcHBzLmNvbS8yZmVnZnU5bHNnNDdpNzF0a2ozcGJsbjR0c29yZGVjNyJdLCJleHAiOjE3ODY4OTU1MjAsImlhdCI6MTc4Njg5MTkyMCwiaXNzIjoiaHR0cHM6Ly9vaWRjLm9wMS5vcGVuc2hpZnRhcHBzLmNvbS8yZmVnZnU5bHNnNDdpNzF0a2ozcGJsbjR0c29yZGVjNyIsImp0aSI6IjNiN2ZlZmIzLWZmNDYtNGQzYS05NGNhLTRhYjIzMWQyMjU4MCIsImt1YmVybmV0ZXMuaW8iOnsibmFtZXNwYWNlIjoidmFnZWVzaHRrLWRldiIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJqZW5raW5zLWRlcGxveWVyIiwidWlkIjoiOGJhY2QwOTktODNlZS00ZWJhLWE0NGMtYmM1M2UyOTA0MWY2In19LCJuYmYiOjE3ODY4OTE5MjAsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDp2YWdlZXNodGstZGV2OmplbmtpbnMtZGVwbG95ZXIifQ.UwTT8beKeLU6DsxBp1bJD4U4ol9NZ6pgZuW6eYS9ar3H_y_aah4vpR_f97Zq5hizMRmXFWw382Pr4wX7ZU6vIDUGKQqsc9FXC1Oh4IxcCRMcneKKSvQG_OiEvNVPL7V7J_LOCfgH1GgSFQPmgqPgOEEsOacfJ7O-kvsLyv-EmupiYH7D_3-VTxCObwV6O5dzjQU6F4KbHvduY17_-mmGnaEVKVbj6fJha1Ti9sGM7vtzV6xQ_zQ24TFWVc-CQHytC9KDXbxYtDcGGzfw8mecvRkVVwfYGah_xdHBr0dHmrHL90ATFcfZ8rCJGuwI9IxkBpXLsaa2s-0xY3IYh3Fc9QFzzH-cQVEuSN1xUtBPihQkEeLivlyLoIfAjcIa5QiNn7cMq3Zfn6i1nIpQGH0IdpsSVG5Ts4XesWidJbE7EPsi6LY19MsFXxOfggP0985LTg_DOVytK0HOpJQ5uIQAQNbTN8LqWW_lY0Gr7y3DsCWOcq4ttPn-XD90pDctynEIXBQoD07xrO0KrkrnbGHFIrE0Foc-XEOWBMga7xO9b5wyU4a-cF5DyykWs0Ty4naXst51bLHTZzjtfCif7Vp5Hvgwrop5FNOh85dXFnpHclJPfejdgxXn77cpchOjB31Gmqu9HGAsHBrJqmBNeCIn_qfdSBUK5dkwxMF5moxfNJQ'

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
