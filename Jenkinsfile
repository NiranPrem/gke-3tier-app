pipeline {
    agent any

    parameters {
        choice(name: 'NAMESPACE', choices: ['dev','staging','prod'], description: 'Target environment for deployment')
    }

    environment {
        PROJECT_ID = "useful-variety-470306-n5"
        REGION = "us-central1"
        CLUSTER = "my-gke-cluster"
        IMAGE_FRONTEND = "gke-3tier-frontend"
        IMAGE_BACKEND = "gke-3tier-backend"
        GCP_CREDS = credentials('gcp-credentials') // This must match your Jenkins credential ID
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        FRONT_IMAGE = "${REGION}-docker.pkg.dev/${PROJECT_ID}/gke-repo/${IMAGE_FRONTEND}:${IMAGE_TAG}"
        BACK_IMAGE = "${REGION}-docker.pkg.dev/${PROJECT_ID}/gke-repo/${IMAGE_BACKEND}:${IMAGE_TAG}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Authenticate GCP') {
            steps {
                script {
                    sh """
                    echo \$GCP_CREDS > /tmp/key.json
                    gcloud auth activate-service-account --key-file=/tmp/key.json
                    gcloud config set project \$PROJECT_ID
                    """
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                script {
                    sh """
                    gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet

                    # Build and push frontend
                    docker build -t \$FRONT_IMAGE ./frontend
                    docker push \$FRONT_IMAGE

                    # Build and push backend
                    docker build -t \$BACK_IMAGE ./backend
                    docker push \$BACK_IMAGE
                    """
                }
            }
        }

        stage('Deploy Green') {
            steps {
                script {
                    sh """
                    gcloud container clusters get-credentials \$CLUSTER --zone us-central1-a --project \$PROJECT_ID

                    # Apply namespace if not exists
                    kubectl apply -f manifests/${params.NAMESPACE}/namespace.yaml || true

                    # Deploy green versions
                    kubectl apply -f manifests/${params.NAMESPACE}/frontend-green.yaml
                    kubectl apply -f manifests/${params.NAMESPACE}/backend-green.yaml

                    # Set images
                    kubectl set image deployment/frontend-green frontend=\$FRONT_IMAGE -n ${params.NAMESPACE}
                    kubectl set image deployment/backend-green backend=\$BACK_IMAGE -n ${params.NAMESPACE}

                    # Wait for rollout
                    kubectl rollout status deployment/frontend-green -n ${params.NAMESPACE}
                    kubectl rollout status deployment/backend-green -n ${params.NAMESPACE}
                    """
                }
            }
        }

        stage('Cutover Blue → Green') {
            steps {
                script {
                    sh """
                    # Switch services to green deployments
                    kubectl -n ${params.NAMESPACE} patch svc frontend-svc --type='json' -p='[{"op":"replace","path":"/spec/selector/color","value":"green"}]'
                    kubectl -n ${params.NAMESPACE} patch svc backend-svc --type='json' -p='[{"op":"replace","path":"/spec/selector/color","value":"green"}]'
                    """
                }
            }
        }

        stage('Cleanup Old Blue') {
            steps {
                script {
                    sh """
                    # Optional: delete blue deployments after cutover
                    kubectl delete deployment frontend-blue -n ${params.NAMESPACE} || true
                    kubectl delete deployment backend-blue -n ${params.NAMESPACE} || true
                    """
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
