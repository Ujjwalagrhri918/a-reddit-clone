# Automation for Blue Green Deployement
```
pipeline {
    agent any
    environment {
        KUBE_NAMESPACE = "default"
        SERVICE_NAME = "reddit-clone-service"
        APP_NAME = "reddit-clone"
        BLUE_IMAGE = "ujjubaby/reddit-clone-pipeline:1.0.0-11"
        GREEN_IMAGE = "ujjubaby/reddit-clone-pipeline:1.0.0-13"
    }
    stages {
        stage('Determine Active Color') {
            steps {
                script {
                    ACTIVE = sh(script: "kubectl get svc ${SERVICE_NAME} -o jsonpath='{.spec.selector.version}'", returnStdout: true).trim()
                    if (ACTIVE == "blue") {
                        NEW_COLOR = "green"
                        NEW_IMAGE = "${GREEN_IMAGE}"
                    } else {
                        NEW_COLOR = "blue"
                        NEW_IMAGE = "${BLUE_IMAGE}"
                    }
                    echo "Active: ${ACTIVE}, Deploying: ${NEW_COLOR} with Image: ${NEW_IMAGE}"
                }
            }
        }
        stage('Deploy New Version') {
            steps {
                sh "kubectl set image deployment/${APP_NAME}-${NEW_COLOR} ${APP_NAME}=${NEW_IMAGE} -n ${KUBE_NAMESPACE}"
                sh "kubectl rollout status deployment/${APP_NAME}-${NEW_COLOR} -n ${KUBE_NAMESPACE}"
            }
        }
        stage('Switch Service') {
            steps {
                sh "kubectl patch svc ${SERVICE_NAME} -p '{\"spec\":{\"selector\":{\"app\":\"${APP_NAME}\",\"version\":\"${NEW_COLOR}\"}}}' -n ${KUBE_NAMESPACE}"
            }
        }
    }
    post {
        success {
            echo "Blue-Green Deployment Successful!"
        }
        failure {
            echo "Deployment Failed! Rollback may be required."
        }
    }
}
```



