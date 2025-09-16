pipeline {
    agent any
    environment {
        PROJECT_ID = 'useful-variety-470306-n5'
        GCP_CREDS = credentials('GCP_CREDS')  // Replace with your Jenkins credential ID
        FRONT_IMAGE = "us-central1-docker.pkg.dev/${PROJECT_ID}/gke-repo/gke-3tier-frontend:latest"
        BACK_IMAGE  = "us-central1-docker.pkg.dev/${PROJECT_ID}/gke-repo/gke-3tier-backend:latest"
        CLUSTER_NAME = 'my-gke-cluster'
        ZONE = 'us-central1-a'
    }

    stages {
        stage('Authenticate GCP') {
            steps {
                withCredentials([file(credentialsId: 'GCP_CREDS', variable: 'GCP_KEY')]) {
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
                script {
                    sh "docker build -t ${FRONT_IMAGE} ./frontend"
                    sh "docker push ${FRONT_IMAGE}"

                    sh "docker build -t ${BACK_IMAGE} ./backend"
                    sh "docker push ${BACK_IMAGE}"
                }
            }
        }

        stage('Deploy Green') {
            steps {
                script {
                    def envs = ['dev', 'staging', 'prod']

                    envs.each { envName ->
                        echo "Deploying green version to ${envName}..."
                        def envDir = "manifests/${envName}"

                        // Get cluster credentials
                        sh "gcloud container clusters get-credentials ${CLUSTER_NAME} --zone ${ZONE} --project ${PROJECT_ID}"

                        // Apply namespace & deployments
                        sh "kubectl apply -f ${envDir}/namespace.yaml"
                        sh "kubectl apply -f ${envDir}/frontend-green.yaml"
                        sh "kubectl apply -f ${envDir}/backend-green.yaml"

                        // Update images
                        sh "kubectl set image deployment/frontend-green frontend=${FRONT_IMAGE} -n ${envName}"
                        sh "kubectl set image deployment/backend-green backend=${BACK_IMAGE} -n ${envName}"
                    }
                }
            }
        }

        stage('Cutover Blue → Green') {
            steps {
                script {
                    def envs = ['dev', 'staging', 'prod']

                    envs.each { envName ->
                        echo "Cutting over ${envName} from blue → green..."
                        sh """
                            kubectl patch svc frontend -n ${envName} -p '{"spec":{"selector":{"app":"frontend-green"}}}'
                            kubectl patch svc backend  -n ${envName} -p '{"spec":{"selector":{"app":"backend-green"}}}'
                        """
                    }
                }
            }
        }

        stage('Cleanup Old Blue') {
            steps {
                script {
                    def envs = ['dev', 'staging', 'prod']

                    envs.each { envName ->
                        echo "Cleaning up old blue deployments in ${envName}..."
                        sh "kubectl delete deployment frontend-blue -n ${envName} || true"
                        sh "kubectl delete deployment backend-blue -n ${envName} || true"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment completed successfully for all environments!"
        }
        failure {
            echo "❌ Deployment failed. Check the logs for details."
        }
        always {
            cleanWs()
        }
    }
}
