pipeline {
    agent any

    environment {
        PROJECT_ID = "useful-variety-470306-n5"
        CLUSTER_NAME = "my-gke-cluster"
        CLUSTER_ZONE = "us-central1-a"
        DOCKER_REPO = "us-central1-docker.pkg.dev/useful-variety-470306-n5/gke-repo"
        IMAGE_TAG = "latest"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Authenticate GCP') {
            steps {
                withCredentials([file(credentialsId: 'GCP_KEY', variable: 'GCP_KEY')]) {
                    sh """
                        gcloud auth activate-service-account --key-file=${GCP_KEY}
                        gcloud config set project ${PROJECT_ID}
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

                            # Create namespace if not exists
                            kubectl apply -f manifests/${env}/namespace.yaml

                            # Delete old green deployments if exist
                            kubectl delete deployment frontend-green -n ${env} --ignore-not-found
                            kubectl delete deployment backend-green -n ${env} --ignore-not-found

                            # Apply green deployments
                            kubectl apply -f manifests/${env}/frontend-deployment.yaml
                            kubectl apply -f manifests/${env}/backend-deployment.yaml

                            # Update images
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
    }

    post {
        success {
            echo "✅ Deployment succeeded!"
        }
        failure {
            echo "❌ Deployment failed. Check the logs."
        }
        always {
            cleanWs()
        }
    }
}
