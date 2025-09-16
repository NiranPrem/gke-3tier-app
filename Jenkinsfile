pipeline {
    agent any

    environment {
        PROJECT_ID    = 'useful-variety-470306-n5'
        REGION        = 'us-central1'
        ZONE          = 'us-central1-a'
        CLUSTER_NAME  = 'my-gke-cluster'
        REPO          = 'gke-repo'
        NAMESPACE_DEV = 'dev'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/NiranPrem/gke-3tier-app.git'
            }
        }

        stage('Authenticate GCP') {
            steps {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
                    sh '''
                        echo "Authenticating with GCP..."
                        gcloud auth activate-service-account --key-file=$GCP_KEY
                        gcloud config set project $PROJECT_ID
                    '''
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                sh """
                    docker build -t $REGION-docker.pkg.dev/$PROJECT_ID/$REPO/gke-3tier-frontend:latest ./frontend
                    docker push $REGION-docker.pkg.dev/$PROJECT_ID/$REPO/gke-3tier-frontend:latest

                    docker build -t $REGION-docker.pkg.dev/$PROJECT_ID/$REPO/gke-3tier-backend:latest ./backend
                    docker push $REGION-docker.pkg.dev/$PROJECT_ID/$REPO/gke-3tier-backend:latest
                """
            }
        }

        stage('Blue-Green Deploy to Dev') {
            steps {
                script {
                    // Determine which is active and idle
                    def activeFrontend = sh(script: "kubectl get svc frontend-svc -n $NAMESPACE_DEV -o jsonpath={.spec.selector.app}", returnStdout: true).trim()
                    def activeBackend  = sh(script: "kubectl get svc backend-svc -n $NAMESPACE_DEV -o jsonpath={.spec.selector.app}", returnStdout: true).trim()

                    def idleFrontend = (activeFrontend == "frontend-blue") ? "frontend-green" : "frontend-blue"
                    def idleBackend  = (activeBackend == "backend-blue") ? "backend-green" : "backend-blue"

                    echo "Active Frontend: ${activeFrontend}, Deploying to: ${idleFrontend}"
                    echo "Active Backend: ${activeBackend}, Deploying to: ${idleBackend}"

                    // Save to environment for later use
                    env.ACTIVE_FRONTEND = activeFrontend
                    env.ACTIVE_BACKEND  = activeBackend
                    env.IDLE_FRONTEND   = idleFrontend
                    env.IDLE_BACKEND    = idleBackend

                    sh """
                        gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT_ID

                        # Apply manifests for idle deployments
                        kubectl apply -f manifests/dev/${idleFrontend}.yaml
                        kubectl set image deployment/${idleFrontend} frontend=$REGION-docker.pkg.dev/$PROJECT_ID/$REPO/gke-3tier-frontend:latest -n $NAMESPACE_DEV

                        kubectl apply -f manifests/dev/${idleBackend}.yaml
                        kubectl set image deployment/${idleBackend} backend=$REGION-docker.pkg.dev/$PROJECT_ID/$REPO/gke-3tier-backend:latest -n $NAMESPACE_DEV
                    """

                    echo "✅ Idle deployments updated. Go test them manually before switching traffic."
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                script {
                    // Pause here for manual confirmation
                    def userChoice = input(
                        id: 'SwitchApproval',
                        message: 'Do you want to switch traffic to the new deployments?',
                        parameters: [
                            choice(name: 'CONFIRM', choices: ['No', 'Yes'], description: 'Switch traffic?')
                        ]
                    )

                    if (userChoice == 'Yes') {
                        sh """
                            kubectl patch svc frontend-svc -n $NAMESPACE_DEV -p '{"spec":{"selector":{"app":"${env.IDLE_FRONTEND}"}}}'
                            kubectl patch svc backend-svc -n $NAMESPACE_DEV -p '{"spec":{"selector":{"app":"${env.IDLE_BACKEND}"}}}'
                        """
                        echo "✅ Traffic switched to new deployments."
                    } else {
                        echo "⚠️ Switch skipped. Traffic still on old deployments."
                    }
                }
            }
        }

        stage('Rollback') {
            steps {
                script {
                    def rollbackChoice = input(
                        id: 'RollbackApproval',
                        message: 'Do you want to rollback traffic to previous deployments?',
                        parameters: [
                            choice(name: 'CONFIRM', choices: ['No', 'Yes'], description: 'Rollback traffic?')
                        ]
                    )

                    if (rollbackChoice == 'Yes') {
                        sh """
                            kubectl patch svc frontend-svc -n $NAMESPACE_DEV -p '{"spec":{"selector":{"app":"${env.ACTIVE_FRONTEND}"}}}'
                            kubectl patch svc backend-svc -n $NAMESPACE_DEV -p '{"spec":{"selector":{"app":"${env.ACTIVE_BACKEND}"}}}'
                        """
                        echo "♻️ Rolled back to previous deployments."
                    } else {
                        echo "✅ No rollback performed."
                    }
                }
            }
        }
    }
}
