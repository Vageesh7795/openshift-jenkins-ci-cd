pipeline {
    agent any

    environment {
        PROJECT = 'cicd-demo'
        APP_NAME = 'myapp'
        IMAGE = 'nginx'
        TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/Vageesh7795/openshift-jenkins-ci-cd.git'
            }
        }

        stage('Build') {
            steps {
                echo "Building application..."
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    oc project ${PROJECT}

                    oc set image deployment/${APP_NAME} \
                    ${APP_NAME}=${IMAGE}:${TAG} \
                    || true

                    oc rollout status deployment/${APP_NAME}
                """
            }
        }

        stage('Verify') {
            steps {
                sh """
                    oc get pods
                    oc get svc
                    oc get route
                """
            }
        }
    }
}
