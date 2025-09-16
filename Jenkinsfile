pipeline {
    agent any
    environment {
        PROJECT_ID = "useful-variety-470306-n5"
        CLUSTER_NAME = "my-gke-cluster"
        ZONE = "us-central1-a"
        GCP_KEY = credentials('GCP_CREDS')
        IMAGE_FRONTEND = "us-central1-docker.pkg.dev/${PROJECT_ID}/gke-repo/gke-3tier-frontend:latest"
        IMAGE_BACKEND  = "us-central1-docker.pkg.dev/${PROJECT_ID}/gke-repo/gke-3tier-backend:latest"
        NAMESPACE_DEV = "dev"
    }
    stages {

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
                sh """
                    docker build -t $IMAGE_FRONTEND ./frontend
                    docker push $IMAGE_FRONTEND

                    docker build -t $IMAGE_BACKEND ./backend
                    docker push $IMAGE_BACKEND
                """
            }
        }

        stage('Blue-Green Deploy to Dev') {
            steps {
                script {
                    // Get currently active deployments
                    ACTIVE_FRONTEND = sh(script: "kubectl get svc frontend-svc -n $NAMESPACE_DEV -o jsonpath='{.spec.selector.app}'", returnStdout: true).trim()
                    ACTIVE_BACKEND  = sh(script: "kubectl get svc backend-svc -n $NAMESPACE_DEV -o jsonpath='{.spec.selector.app}'", returnStdout: true).trim()

                    // Determine idle deployments
                    IDLE_FRONTEND = (ACTIVE_FRONTEND == "frontend-blue") ? "frontend-green" : "frontend-blue"
                    IDLE_BACKEND  = (ACTIVE_BACKEND == "backend-blue") ? "backend-green" : "backend-blue"

                    echo "Active Frontend: ${ACTIVE_FRONTEND}, Deploying to: ${IDLE_FRONTEND}"
                    echo "Active Backend: ${ACTIVE_BACKEND}, Deploying to: ${IDLE_BACKEND}"

                    // Deploy to idle
                    sh """
                        gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT_ID

                        kubectl apply -f manifests/dev/${IDLE_FRONTEND}.yaml
                        kubectl set image deployment/${IDLE_FRONTEND} frontend=$IMAGE_FRONTEND -n $NAMESPACE_DEV

                        kubectl apply -f manifests/dev/${IDLE_BACKEND}.yaml
                        kubectl set image deployment/${IDLE_BACKEND} backend=$IMAGE_BACKEND -n $NAMESPACE_DEV
                    """

                    // Switch services to new deployments
                    sh """
                        kubectl patch svc frontend-svc -n $NAMESPACE_DEV -p '{"spec":{"selector":{"app":"${IDLE_FRONTEND}"}}}'
                        kubectl patch svc backend-svc -n $NAMESPACE_DEV -p '{"spec":{"selector":{"app":"${IDLE_BACKEND}"}}}'
                    """
                }
            }
        }

        stage('Cleanup Old Deployment') {
            steps {
                script {
                    sh """
                        kubectl delete deployment $ACTIVE_FRONTEND -n $NAMESPACE_DEV --ignore-not-found
                        kubectl delete deployment $ACTIVE_BACKEND -n $NAMESPACE_DEV --ignore-not-found
                    """
                }
            }
        }

    } // stages
}
