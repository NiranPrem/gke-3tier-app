pipeline {
    agent any

    environment {
        PROJECT_ID = "useful-variety-470306-n5"
        REGION = "us-central1"
        CLUSTER = "my-gke-cluster"
        IMAGE_FRONTEND = "gke-3tier-frontend"
        IMAGE_BACKEND = "gke-3tier-backend"
        GCP_CREDS = credentials('gke-sa-keys') // your Jenkins GCP service account JSON
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/NiranPrem/gke-3tier-app.git'
            }
        }

        stage('Build & Push Images') {
            steps {
                script {
                    sh """
                      echo \$GCP_CREDS > /tmp/key.json
                      gcloud auth activate-service-account --key-file=/tmp/key.json
                      gcloud config set project \$PROJECT_ID

                      # Build frontend
                      docker build -t gcr.io/\$PROJECT_ID/\$IMAGE_FRONTEND:\$BUILD_NUMBER ./frontend
                      docker push gcr.io/\$PROJECT_ID/\$IMAGE_FRONTEND:\$BUILD_NUMBER

                      # Build backend
                      docker build -t gcr.io/\$PROJECT_ID/\$IMAGE_BACKEND:\$BUILD_NUMBER ./backend
                      docker push gcr.io/\$PROJECT_ID/\$IMAGE_BACKEND:\$BUILD_NUMBER
                    """
                }
            }
        }

        stage('Deploy to Dev') {
            steps {
                script {
                    sh """
                      gcloud container clusters get-credentials \$CLUSTER --zone \$REGION --project \$PROJECT_ID

                      kubectl set image deployment/frontend frontend=gcr.io/\$PROJECT_ID/\$IMAGE_FRONTEND:\$BUILD_NUMBER -n dev
                      kubectl set image deployment/backend backend=gcr.io/\$PROJECT_ID/\$IMAGE_BACKEND:\$BUILD_NUMBER -n dev
                    """
                }
            }
        }

        stage('Manual Approval for Staging') {
            steps {
                input "Deploy to Staging?"
            }
        }

        stage('Deploy to Staging (Blue/Green)') {
            steps {
                script {
                    sh """
                      gcloud container clusters get-credentials \$CLUSTER --zone \$REGION --project \$PROJECT_ID

                      kubectl apply -f k8s/staging/ -n staging
                      kubectl set image deployment/frontend frontend=gcr.io/\$PROJECT_ID/\$IMAGE_FRONTEND:\$BUILD_NUMBER -n staging
                      kubectl set image deployment/backend backend=gcr.io/\$PROJECT_ID/\$IMAGE_BACKEND:\$BUILD_NUMBER -n staging
                    """
                }
            }
        }

        stage('Manual Approval for Prod') {
            steps {
                input "Deploy to Prod (Blue/Green)?"
            }
        }

        stage('Deploy to Prod (Blue/Green)') {
            steps {
                script {
                    sh """
                      gcloud container clusters get-credentials \$CLUSTER --zone \$REGION --project \$PROJECT_ID

                      kubectl apply -f k8s/prod/ -n prod
                      kubectl set image deployment/frontend frontend=gcr.io/\$PROJECT_ID/\$IMAGE_FRONTEND:\$BUILD_NUMBER -n prod
                      kubectl set image deployment/backend backend=gcr.io/\$PROJECT_ID/\$IMAGE_BACKEND:\$BUILD_NUMBER -n prod
                    """
                }
            }
        }
    }
}

