pipeline {
    agent any

    environment {
        PROJECT_ID = 'useful-variety-470306-n5'
        CLUSTER_NAME = 'my-gke-cluster'
        ZONE = 'us-central1-a'
        FRONTEND_IMAGE = "us-central1-docker.pkg.dev/${PROJECT_ID}/gke-repo/gke-3tier-frontend:latest"
        BACKEND_IMAGE = "us-central1-docker.pkg.dev/${PROJECT_ID}/gke-repo/gke-3tier-backend:latest"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                git url: 'https://github.com/NiranPrem/gke-3tier-app.git', branch: 'main'
            }
        }

        stage('Authenticate GCP') {
            steps {
                withCredentials([file(credentialsId: 'GCP_CREDS', variable: 'GCP_KEY')]) {
                    sh '''
                        echo "Authenticating with GCP..."
                        gcloud auth activate-service-account --key-file=${GCP_KEY}
                        gcloud config set project ${PROJECT_ID}
                    '''
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                sh """
                    docker build -t ${FRONTEND_IMAGE} ./frontend
                    docker push ${FRONTEND_IMAGE}

                    docker build -t ${BACKEND_IMAGE} ./backend
                    docker push ${BACKEND_IMAGE}
                """
            }
        }

        stage('Deploy Green') {
            steps {
                script {
                    for (envName in ['dev', 'staging', 'prod']) {
                        echo "Deploying green version to ${envName}..."
                        sh """
                            gcloud container clusters get-credentials ${CLUSTER_NAME} --zone ${ZONE} --project ${PROJECT_ID}

                            # Create namespace if not exists
                            kubectl apply -f manifests/${envName}/namespace.yaml

                            # Delete old green deployments if exist
                            kubectl delete deployment frontend-green -n ${envName} --ignore-not-found
                            kubectl delete deployment backend-green -n ${envName} --ignore-not-found

                            # Apply new green deployments
                            kubectl apply -f manifests/${envName}/frontend-green.yaml
                            kubectl apply -f manifests/${envName}/backend-green.yaml

                            # Update images for deployments
                            kubectl set image deployment/frontend-green frontend=${FRONTEND_IMAGE} -n ${envName}
                            kubectl set image deployment/backend-green backend=${BACKEND_IMAGE} -n ${envName}
                        """
                    }
                }
            }
        }

        stage('Cutover Blue → Green') {
            steps {
                script {
                    for (envName in ['dev', 'staging', 'prod']) {
                        echo "Cutting over ${envName} from blue → green..."
                        sh """
                            gcloud container clusters get-credentials ${CLUSTER_NAME} --zone ${ZONE} --project ${PROJECT_ID}

                            kubectl patch svc frontend -n ${envName} -p '{"spec":{"selector":{"app":"frontend-green"}}}'
                            kubectl patch svc backend -n ${envName} -p '{"spec":{"selector":{"app":"backend-green"}}}'
                        """
                    }
                }
            }
        }

        stage('Cleanup Old Blue') {
            steps {
                script {
                    for (envName in ['dev', 'staging', 'prod']) {
                        echo "Cleaning up old blue deployments in ${envName}..."
                        sh """
                            kubectl delete deployment frontend-blue -n ${envName} --ignore-not-found
                            kubectl delete deployment backend-blue -n ${envName} --ignore-not-found
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            echo "✅ Pipeline finished"
        }
        failure {
            echo "❌ Deployment failed. Check logs."
        }
    }
}
