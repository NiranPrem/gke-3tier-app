pipeline {
    agent any

    parameters {
        choice(name: 'NAMESPACE', choices: ['dev','staging','prod'], description: 'Select environment to deploy')
    }

    environment {
        PROJECT_ID = "useful-variety-470306-n5"
        REGION = "us-central1"
        CLUSTER = "my-gke-cluster"
        IMAGE_FRONTEND = "gke-3tier-frontend"
        IMAGE_BACKEND = "gke-3tier-backend"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Authenticate GCP') {
            steps {
                withCredentials([file(credentialsId: 'gcp-credentials', variable: 'GCP_CREDS')]) {
                    sh '''
                        echo "Authenticating with GCP..."
                        gcloud auth activate-service-account --key-file=$GCP_CREDS
                        gcloud config set project $PROJECT_ID
                    '''
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                script {
                    sh '''
                        echo "Building and pushing Docker images..."

                        gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet

                        FRONT_IMAGE=${REGION}-docker.pkg.dev/${PROJECT_ID}/gke-repo/${IMAGE_FRONTEND}:${BUILD_NUMBER}
                        BACK_IMAGE=${REGION}-docker.pkg.dev/${PROJECT_ID}/gke-repo/${IMAGE_BACKEND}:${BUILD_NUMBER}

                        echo "Frontend image: $FRONT_IMAGE"
                        echo "Backend image: $BACK_IMAGE"

                        # Build and push frontend
                        docker build -t $FRONT_IMAGE ./frontend
                        docker push $FRONT_IMAGE

                        # Build and push backend
                        docker build -t $BACK_IMAGE ./backend
                        docker push $BACK_IMAGE
                    '''
                }
            }
        }

        stage('Deploy Green') {
            steps {
                script {
                    sh '''
                        echo "Deploying green versions..."

                        gcloud container clusters get-credentials $CLUSTER --zone us-central1-a --project $PROJECT_ID

                        # Create namespace if not exists
                        kubectl apply -f manifests/${NAMESPACE}/namespace.yaml || true

                        # Deploy green versions
                        kubectl apply -f manifests/${NAMESPACE}/frontend-green.yaml
                        kubectl apply -f manifests/${NAMESPACE}/backend-green.yaml

                        # Set images for green deployments
                        kubectl set image deployment/frontend-green frontend=$FRONT_IMAGE -n ${NAMESPACE}
                        kubectl set image deployment/backend-green backend=$BACK_IMAGE -n ${NAMESPACE}

                        # Wait for rollout
                        kubectl rollout status deployment/frontend-green -n ${NAMESPACE}
                        kubectl rollout status deployment/backend-green -n ${NAMESPACE}
                    '''
                }
            }
        }

        stage('Cutover Blue → Green') {
            steps {
                script {
                    sh '''
                        echo "Switching services from blue to green..."
                        kubectl -n ${NAMESPACE} patch svc frontend-svc --type='json' -p='[{"op":"replace","path":"/spec/selector/color","value":"green"}]'
                        kubectl -n ${NAMESPACE} patch svc backend-svc --type='json' -p='[{"op":"replace","path":"/spec/selector/color","value":"green"}]'
                    '''
                }
            }
        }

        stage('Cleanup Old Blue') {
            steps {
                script {
                    sh '''
                        echo "Cleaning up old blue deployments..."
                        kubectl delete deployment frontend-blue -n ${NAMESPACE} || true
                        kubectl delete deployment backend-blue -n ${NAMESPACE} || true
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
