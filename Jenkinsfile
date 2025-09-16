pipeline {
    agent any

    environment {
        PROJECT_ID = 'useful-variety-470306-n5'
        CLUSTER_ZONE = 'us-central1-a'
        CLUSTER_NAME = 'my-gke-cluster'
        GCP_KEY = credentials('GCP_CREDS') // Jenkins GCP service account
        IMAGE_TAG = "latest"
        DOCKER_REPO = "us-central1-docker.pkg.dev/${PROJECT_ID}/gke-repo"
    }

    stages {
        stage('Authenticate GCP') {
            steps {
                withCredentials([file(credentialsId: 'GCP_CREDS', variable: 'GCP_KEY')]) {
                    sh """
                        echo Authenticating with GCP...
                        gcloud auth activate-service-account --key-file=$GCP_KEY
                        gcloud config set project $PROJECT_ID
                    """
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                script {
                    sh """
                        docker build -t ${DOCKER_REPO}/gke-3tier-frontend:${IMAGE_TAG} ./frontend
                        docker push ${DOCKER_REPO}/gke-3tier-frontend:${IMAGE_TAG}

                        docker build -t ${DOCKER_REPO}/gke-3tier-backend:${IMAGE_TAG} ./backend
                        docker push ${DOCKER_REPO}/gke-3tier-backend:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy Green') {
            steps {
                script {
                    for (env in ['dev','staging','prod']) {
                        echo "Deploying green version to ${env}..."
                        sh """
                            gcloud container clusters get-credentials ${CLUSTER_NAME} --zone ${CLUSTER_ZONE} --project ${PROJECT_ID}
                            kubectl apply -f manifests/${env}/namespace.yaml
                            kubectl apply -f manifests/${env}/frontend-green.yaml
                            kubectl apply -f manifests/${env}/backend-green.yaml
                            kubectl set image deployment/frontend-green frontend=${DOCKER_REPO}/gke-3tier-frontend:${IMAGE_TAG} -n ${env}
                            kubectl set image deployment/backend-green backend=${DOCKER_REPO}/gke-3tier-backend:${IMAGE_TAG} -n ${env}
                        """
                    }
                }
            }
        }

        stage('Cutover Blue → Green') {
            steps {
                script {
                    for (env in ['dev','staging','prod']) {
                        echo "Cutting over ${env} from blue → green..."
                        sh """
                            kubectl patch svc frontend -n ${env} -p '{"spec":{"selector":{"app":"frontend-green"}}}'
                            kubectl patch svc backend -n ${env} -p '{"spec":{"selector":{"app":"backend-green"}}}'
                        """
                    }
                }
            }
        }

        stage('Cleanup Old Blue') {
            steps {
                script {
                    for (env in ['dev','staging','prod']) {
                        echo "Cleaning up old blue deployments in ${env}..."
                        sh """
                            kubectl delete deployment frontend-blue -n ${env} || true
                            kubectl delete deployment backend-blue -n ${env} || true
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful!"
        }
        failure {
            echo "❌ Deployment failed. Check logs."
        }
    }
}
