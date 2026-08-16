pipeline {
    agent any

    environment {
        PROJECT = 'vageeshtk-dev'
        APP_NAME = 'my-application'

        // Change this to your OpenShift API server
        OC_SERVER = 'https://172.30.0.1:443'

        // Jenkins ServiceAccount token
        OC_TOKEN = 'eyJhbGciOiJSUzI1NiIsImtpZCI6IkZYZUc1d1FpY1BJV0lfNlNjRUhRM1hHOHE0NWd0X2cxY1o0TXUwQV9lMWcifQ.eyJhdWQiOlsiaHR0cHM6Ly9vaWRjLm9wMS5vcGVuc2hpZnRhcHBzLmNvbS8yZmVnZnU5bHNnNDdpNzF0a2ozcGJsbjR0c29yZGVjNyJdLCJleHAiOjE3ODY4OTE2MjcsImlhdCI6MTc4Njg4ODAyNywiaXNzIjoiaHR0cHM6Ly9vaWRjLm9wMS5vcGVuc2hpZnRhcHBzLmNvbS8yZmVnZnU5bHNnNDdpNzF0a2ozcGJsbjR0c29yZGVjNyIsImp0aSI6IjhjYWJhMzkwLTcxNTgtNDU5NS05ZGU1LWQ0ZDQzZmI5MWNiYSIsImt1YmVybmV0ZXMuaW8iOnsibmFtZXNwYWNlIjoidmFnZWVzaHRrLWRldiIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJqZW5raW5zLWRlcGxveWVyIiwidWlkIjoiOGJhY2QwOTktODNlZS00ZWJhLWE0NGMtYmM1M2UyOTA0MWY2In19LCJuYmYiOjE3ODY4ODgwMjcsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDp2YWdlZXNodGstZGV2OmplbmtpbnMtZGVwbG95ZXIifQ.Q7INE1XYWU3_3B9yU2wm9prv9wihFZxkezygMX14XxqFPveIgyR-EcKEoy-M9KAc8Z64ORRJSCPF2sX-h_dVAC5l-Q4HXx85e4HLKOXEP1iq2DJvXVzqd-HPANhmDdXIKPrPjjBNfebLY6UNtWBD5MedvhP55w9MA9B3n2kQlUdYU3k4EVLGHNu60SJLJl0i1ago3Dq7H27tM1z7OQvAPk9aFwMUtpgAZxWSeeuhr2tl9uqYWFyN4LIggIUf8rxFFuZtwxmf5KVZSC0C5K4EGJ0k4N2DfkzK3vi80xUFRyfiWlP3vzvx6q6uDMev9Ta6Wdm8S6nO9Ym4aDrgakPOAF1oATIj0PkGCp6MYu-F_Bps0dxwUFBXksJI5p9vfvONj1-H34UNUeJTqRHtQ69EDK5-hax8fksfkInEeUEMDMKAWTHBtfmcb796PSul0NFGQP0j2ZAUAc8nbjgZm2JYRywsiiOfKWdBgHtBybne0TvRv9lWVB3yHXgzQidSxtiqieXHMaKeVE0S9Kd06nYNdIO2wG78_oYkxC8YwOP1lqkv-T-I-1qvORDAr5MJPpOCYi6sr1Nd0kUVvq8SMjawQVN-FujN7au6WYkywCJTJZIB_1YhpwI_zXHQjSlLOrtF_ssPLaBWapwk4nhNXspd84LPYYaBFL39CBE1W_j_zJE'

        REGISTRY = 'image-registry.openshift-image-registry.svc:5000'
        IMAGE = "${REGISTRY}/${PROJECT}/${APP_NAME}"
        TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
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

        stage('Build Image') {
            steps {
                sh '''
                    echo "Building Docker image..."

                    podman build \
                      -t "${IMAGE}:${TAG}" \
                      -t "${IMAGE}:latest" \
                      .

                    echo "Images created:"
                    /usr/bin/podman images | grep "${APP_NAME}"
                '''
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    echo "Logging into OpenShift registry..."

                    oc registry login

                    echo "Pushing image: ${IMAGE}:${TAG}"

                    /usr/bin/podman push "${IMAGE}:${TAG}"

                    echo "Pushing latest image..."

                    /usr/bin/podman push "${IMAGE}:latest"

                    echo "Image pushed successfully."
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

                    echo "Deployment completed."
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    echo "========== PODS =========="
                    oc get pods -o wide

                    echo "========== DEPLOYMENT =========="
                    oc get deployment ${APP_NAME}

                    echo "========== SERVICE =========="
                    oc get svc ${APP_NAME}

                    echo "========== ROUTE =========="
                    oc get route ${APP_NAME}

                    echo "========== APPLICATION URL =========="
                    oc get route ${APP_NAME} \
                      -o jsonpath='{.spec.host}'
                    echo
                '''
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo ' APPLICATION DEPLOYED SUCCESSFULLY'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo ' DEPLOYMENT FAILED'
            echo ' Check Jenkins console output'
            echo '======================================'
        }
    }
}
