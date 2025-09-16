pipeline {
    agent any

    environment {
        PROJECT_ID = 'useful-variety-470306-n5'
        CLUSTER_NAME = 'my-gke-cluster'
        ZONE = 'us-central1-a'
        GCP_KEY = credentials('GCP_CREDS')  // your Jenkins credential ID
        DOCKER_REPO = "us-central1-docker.pkg.dev/${PROJECT_ID}/gke-repo"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Authenticate GCP') {
            steps {
                withCredentials([file(credentialsId: 'GCP_CREDS', variable: 'GCP_KEY')]) {
                    sh """
                        echo "Authenticating with GCP..."
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
                        # Frontend
                        docker build -t ${DOCKER_REPO}/gke-3tier-frontend:latest ./frontend
                        docker push ${DOCKER_REPO}/gke-3tier-frontend:latest

                        # Backend
                        docker build -t ${DOCKER_REPO}/gke-3tier-backend:latest ./backend
                        docker push ${DOCKER_REPO}/gke-3tier-backend:latest
                    """
                }
            }
        }

        stage('Deploy Green') {
            steps {
                script {
                    def environments = ['dev', 'staging', 'prod']

                    for (envName in environments) {
                        echo "Deploying green version to ${envName}..."

                        sh """
                            # Get GKE credentials
                            gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT_ID

                            # Create namespace if not exists
                            kubectl apply -f manifests/${envName}/namespace.yaml

                            # Delete old green deployments if any
                            kubectl delete deployment frontend-green -n ${envName} --ignore-not-found
                            kubectl delete deployment backend-green -n ${envName} --ignore-not-found

                            # Apply new green deployments
                            kubectl apply -f manifests/${envName}/frontend-green.yaml
                            kubectl apply -f manifests/${envName}/backend-green.yaml

                            # Apply services (needed for cutover)
                            kubectl apply -f manifests/${envName}/svc-frontend.yaml
                            kubectl apply -f manifests/${envName}/svc-backend.yaml

                            # Update deployment images to latest
                            kubectl set image deployment/frontend-green frontend=${DOCKER_REPO}/gke-3tier-frontend:latest -n ${envName}
                            kubectl set image deployment/backend-green backend=${DOCKER_REPO}/gke-3tier-backend:latest -n ${envName}
                        """
                    }
                }
            }
        }

        stage('Cutover Blue → Green') {
            steps {
                script {
                    def environments = ['dev', 'staging', 'prod']

                    for (envName in environments) {
                        echo "Cutting over ${envName} from blue → green..."

                        sh """
                            gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT_ID

                            # Patch frontend service to green
                            kubectl patch svc frontend -n ${envName} -p '{"spec":{"selector":{"app":"frontend-green"}}}' || echo "Service frontend not found"

                            # Patch backend service to green
                            kubectl patch svc backend -n ${envName} -p '{"spec":{"selector":{"app":"backend-green"}}}' || echo "Service backend not found"
                        """
                    }
                }
            }
        }

        stage('Cleanup Old Blue') {
            steps {
                script {
                    def environments = ['dev', 'staging', 'prod']

                    for (envName in environments) {
                        echo "Cleaning up old blue deployments in ${envName}..."

                        sh """
                            gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT_ID
                            kubectl delete deployment frontend-blue -n ${envName} --ignore-not-found
                            kubectl delete deployment backend-blue -n ${envName} --ignore-not-found
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline finished successfully"
        }
        failure {
            echo "❌ Deployment failed. Check logs."
        }
        always {
            cleanWs()
        }
    }
}
