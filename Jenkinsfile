pipeline {
    agent any
    environment {
        PROJECT_ID     = "useful-variety-470306-n5"
        CLUSTER_NAME   = "my-gke-cluster"
        ZONE           = "us-central1-a"
        GCP_KEY        = credentials('GCP_CREDS')
        IMAGE_FRONTEND = "us-central1-docker.pkg.dev/${PROJECT_ID}/gke-repo/gke-3tier-frontend:latest"
        IMAGE_BACKEND  = "us-central1-docker.pkg.dev/${PROJECT_ID}/gke-repo/gke-3tier-backend:latest"
        NAMESPACE_DEV  = "dev"

        // SonarQube settings
        SONAR_PROJECT_KEY = "mysonar"        // Your SonarQube project key
        SONAR_SCANNER     = "SonarScanner"   // Tool name as configured in Jenkins
    }
    stages {

        stage('SCM Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER}"
                    withSonarQubeEnv('MySonarQubeServer') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=${SONAR_PROJECT_KEY} -Dsonar.sources=."
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    // Wait up to 5 minutes for SonarQube to compute quality gate
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Authenticate GCP') {
            steps {
                withCredentials([file(credentialsId: 'GCP_CREDS', variable: 'GCP_KEY')]) {
                    sh """
                        echo "Authenticating with GCP..."
                        gcloud auth activate-service-account --key-file=$GCP_KEY
                        gcloud config set project $PROJECT_ID
                        gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT_ID
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

        stage('Apply Ingress') {
            steps {
                sh "kubectl apply -f manifests/dev/ingress.yaml"
            }
        }

        stage('Deploy to Idle (Blue/Green)') {
            steps {
                script {
                    ACTIVE_FRONTEND = sh(script: "kubectl get svc frontend-svc -n $NAMESPACE_DEV -o jsonpath='{.spec.selector.app}'", returnStdout: true).trim()
                    ACTIVE_BACKEND  = sh(script: "kubectl get svc backend-svc -n $NAMESPACE_DEV -o jsonpath='{.spec.selector.app}'", returnStdout: true).trim()

                    IDLE_FRONTEND = (ACTIVE_FRONTEND == "frontend-blue") ? "frontend-green" : "frontend-blue"
                    IDLE_BACKEND  = (ACTIVE_BACKEND == "backend-blue") ? "backend-green" : "backend-blue"

                    echo "Active Frontend: ${ACTIVE_FRONTEND}, Deploying to: ${IDLE_FRONTEND}"
                    echo "Active Backend: ${ACTIVE_BACKEND}, Deploying to: ${IDLE_BACKEND}"

                    sh """
                        kubectl apply -f manifests/dev/${IDLE_FRONTEND}.yaml
                        kubectl set image deployment/${IDLE_FRONTEND} frontend=$IMAGE_FRONTEND -n $NAMESPACE_DEV

                        kubectl apply -f manifests/dev/${IDLE_BACKEND}.yaml
                        kubectl set image deployment/${IDLE_BACKEND} backend=$IMAGE_BACKEND -n $NAMESPACE_DEV
                    """
                }
            }
        }

        stage('Approval Gate: Switch Traffic') {
            steps {
                input(
                  id: 'SwitchApproval',
                  message: "Do you want to switch traffic to the new deployments?",
                  ok: "Switch",
                  submitter: "niranprem"
                )
            }
        }

        stage('Switch Traffic') {
            steps {
                script {
                    sh """
                        kubectl patch svc frontend-svc -n $NAMESPACE_DEV -p '{"spec":{"selector":{"app":"${IDLE_FRONTEND}"}}}'
                        kubectl patch svc backend-svc -n $NAMESPACE_DEV -p '{"spec":{"selector":{"app":"${IDLE_BACKEND}"}}}'
                    """
                }
            }
        }

        stage('Approval Gate: Cleanup Old') {
            steps {
                input(
                  id: 'CleanupApproval',
                  message: "Do you want to delete the old deployments now?",
                  ok: "Delete",
                  submitter: "niranprem"
                )
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
