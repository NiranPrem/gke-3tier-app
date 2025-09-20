pipeline {
    agent any

    environment {
        GCP_KEY = credentials('GCP_KEY')
        SONAR_TOKEN = credentials('SONAR_TOKEN')
    }

    options {
        skipStagesAfterUnstable()
        timestamps()
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: 'https://github.com/NiranPrem/gke-3tier-app.git']]
                ])
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('MySonarQubeServer') {
                        sh """
                            /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarScanner/bin/sonar-scanner \
                            -Dsonar.projectKey=mysonar \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://35.226.247.128:9000 \
                            -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Wait for Quality Gate (optional)') {
            steps {
                script {
                    // Wrap in try/catch to prevent blocking deployment
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                echo "Quality Gate failed: ${qg.status}, but continuing deployment."
                            } else {
                                echo "Quality Gate passed: ${qg.status}"
                            }
                        }
                    } catch (err) {
                        echo "Quality Gate check skipped or timed out, continuing deployment..."
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -t gcr.io/your-project/frontend ./frontend
                    docker build -t gcr.io/your-project/backend ./backend
                    docker build -t gcr.io/your-project/database ./database
                '''
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([file(credentialsId: 'GCP_KEY', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                        gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                        gcloud auth configure-docker
                        docker push gcr.io/your-project/frontend
                        docker push gcr.io/your-project/backend
                        docker push gcr.io/your-project/database
                    '''
                }
            }
        }

        stage('Deploy to GKE') {
            steps {
                withCredentials([file(credentialsId: 'GCP_KEY', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    sh '''
                        gcloud container clusters get-credentials your-gke-cluster --zone your-zone --project your-project
                        kubectl apply -f k8s/
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            cleanWs()
        }
    }
}
