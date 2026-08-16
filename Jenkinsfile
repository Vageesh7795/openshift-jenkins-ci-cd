pipeline {
    agent any

    environment {
        PROJECT = 'cicd-demo'
        APP_NAME = 'my-application'
        REGISTRY = 'image-registry.openshift-image-registry.svc:5000'
        IMAGE = "${REGISTRY}/${PROJECT}/${APP_NAME}"
        TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
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
                        oc login --token=${OC_TOKEN} \
                          --server=${OC_SERVER} \
                          --insecure-skip-tls-verify=true

                        oc project ${PROJECT}
                    '''
                }
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    podman build \
                      -t ${IMAGE}:${TAG} \
                      -t ${IMAGE}:latest \
                      .
                '''
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    oc registry login

                    podman push \
                      ${IMAGE}:${TAG}

                    podman push \
                      ${IMAGE}:latest
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    oc apply -f k8s/deployment.yaml
                    oc apply -f k8s/service.yaml
                    oc apply -f k8s/route.yaml

                    oc set image deployment/${APP_NAME} \
                      ${APP_NAME}=${IMAGE}:${TAG}

                    oc rollout status deployment/${APP_NAME}
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    echo "Pods:"
                    oc get pods

                    echo "Service:"
                    oc get svc ${APP_NAME}

                    echo "Route:"
                    oc get route ${APP_NAME}
                '''
            }
        }
    }

    post {
        success {
            echo "Application deployed successfully!"
        }

        failure {
            echo "Deployment failed!"
        }
    }
}
