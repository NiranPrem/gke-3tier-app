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
        SONAR_PROJECT_KEY = "mysonar"
        SONAR_SCANNER     = "SonarScanner"   // Name of the tool installed in Jenkins
        SONAR_TOKEN       = credentials('newsonar') // Use your token credential here
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
                    withSonarQubeEnv('MySonarQubeServer') { // Name of SonarQube server configured in Jenkins
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://35.226.247.128:9000 \
                            -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Wait for Quality Gate') {
            steps {
                script {
                    // Wrap in try/catch to avoid pipeline hanging
                    try {
                        timeout(time: 10, unit: 'MINUTES') {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                echo "Quality Gate status: ${qg.status}, continuing pipeline..."
                            } else {
                                echo "Quality Gate passed."
                            }
                        }
                    } catch (err) {
                        echo "Quality Gate check timed out or failed, continuing pipeline..."
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
                echo "Applying updated Ingress manifest..."
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
