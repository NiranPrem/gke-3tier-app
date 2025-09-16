pipeline {
    agent any

    environment {
        PROJECT_ID = 'useful-variety-470306-n5'
        ZONE = 'us-central1-a'
        CLUSTER_NAME = 'my-gke-cluster'
        REPO = 'us-central1-docker.pkg.dev/useful-variety-470306-n5/gke-repo'

        // Docker image tags using BUILD_NUMBER
        FRONT_IMAGE = "${REPO}/gke-3tier-frontend:${BUILD_NUMBER}"
        BACK_IMAGE  = "${REPO}/gke-3tier-backend:${BUILD_NUMBER}"
    }

    parameters {
        choice(name: 'ENV', choices: ['dev','staging','prod'], description: 'Select environment')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Authenticate GCP') {
            steps {
                withCredentials([file(credentialsId: 'GCP_CREDS', variable: 'GCP_CREDS')]) {
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
                sh '''
                    echo "Building frontend image $FRONT_IMAGE"
                    docker build -t $FRONT_IMAGE ./frontend
                    docker push $FRONT_IMAGE

                    echo "Building backend image $BACK_IMAGE"
                    docker build -t $BACK_IMAGE ./backend
                    docker push $BACK_IMAGE
                '''
            }
        }

        stage('Deploy Green') {
            steps {
                script {
                    def envDir = "manifests/${params.ENV}"

                    sh """
                        echo "Deploying green version to ${params.ENV}..."
                        gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT_ID

                        # Apply namespace
                        kubectl apply -f ${envDir}/namespace.yaml

                        # Apply green deployments
                        kubectl apply -f ${envDir}/frontend-green.yaml
                        kubectl apply -f ${envDir}/backend-green.yaml

                        # Update deployments with new images
                        kubectl set image deployment/frontend-green frontend=$FRONT_IMAGE -n ${params.ENV}
                        kubectl set image deployment/backend-green backend=$BACK_IMAGE -n ${params.ENV}
                    """
                }
            }
        }

        stage('Cutover Blue → Green') {
            steps {
                script {
                    sh """
                        echo "Switching traffic from blue to green in ${params.ENV}..."
                        envDir="manifests/${params.ENV}"

                        # Update service selectors to green pods
                        kubectl apply -f ${envDir}/frontend-svc-green.yaml
                        kubectl apply -f ${envDir}/backend-svc-green.yaml
                    """
                }
            }
        }

        stage('Cleanup Old Blue') {
            steps {
                script {
                    sh """
                        echo "Cleaning up old blue deployments in ${params.ENV}..."
                        envDir="manifests/${params.ENV}"

                        kubectl delete -f ${envDir}/frontend-blue.yaml --ignore-not-found
                        kubectl delete -f ${envDir}/backend-blue.yaml --ignore-not-found
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment to ${params.ENV} succeeded!"
        }
        failure {
            echo "Deployment to ${params.ENV} failed!"
        }
        always {
            cleanWs()
        }
    }
}
