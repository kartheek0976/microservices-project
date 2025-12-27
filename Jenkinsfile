pipeline {
    agent any

    environment {
        APP_NAME = "adservice"
        IMAGE_NAME = "kartheek0976/adservice"
        REGISTRY_CREDENTIALS = "docker-cred"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        /* --------------------------------------------------
         * COMMON STAGES – ALL BRANCHES
         * -------------------------------------------------- */

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build & Unit Tests') {
            steps {
                sh 'mvn clean test'
            }
        }

        stage('Static Code Analysis (SAST)') {
            steps {
                sh 'mvn sonar:sonar'
            }
        }

        stage('Dependency Scan (SCA)') {
            steps {
                sh '''
                  trivy fs \
                  --severity HIGH,CRITICAL \
                  --no-progress \
                  .
                '''
            }
        }

        /* --------------------------------------------------
         * DEVELOP BRANCH – STAGING FLOW
         * -------------------------------------------------- */

        stage('Build Docker Image (Develop)') {
            when { branch 'develop' }
            steps {
                sh "docker build -t ${IMAGE_NAME}:develop ."
            }
        }

        stage('Image Scan (Develop)') {
            when { branch 'develop' }
            steps {
                sh "trivy image ${IMAGE_NAME}:develop"
            }
        }

        stage('Push Image (Develop)') {
            when { branch 'develop' }
            steps {
                script {
                    withDockerRegistry(
                        credentialsId: REGISTRY_CREDENTIALS,
                        toolName: 'docker'
                    ) {
                        sh "docker push ${IMAGE_NAME}:develop"
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            when { branch 'develop' }
            steps {
                sh 'kubectl apply -f k8s/staging/'
            }
        }

        /* --------------------------------------------------
         * MAIN / MASTER – PRODUCTION FLOW
         * -------------------------------------------------- */

        stage('Build Docker Image (Prod)') {
            when { branch 'main' }
            steps {
                sh "docker build -t ${IMAGE_NAME}:prod ."
            }
        }

        stage('Trivy Image Scan (Prod – Block on Vulns)') {
            when { branch 'main' }
            steps {
                sh '''
                  trivy image \
                  --exit-code 1 \
                  --severity HIGH,CRITICAL \
                  --no-progress \
                  kartheek0976/adservice:prod
                '''
            }
        }

        stage('Kubernetes Config Scan') {
            when { branch 'main' }
            steps {
                sh 'trivy config k8s/prod/'
            }
        }

        stage('Push Image (Prod)') {
            when { branch 'main' }
            steps {
                script {
                    withDockerRegistry(
                        credentialsId: REGISTRY_CREDENTIALS,
                        toolName: 'docker'
                    ) {
                        sh "docker push ${IMAGE_NAME}:prod"
                    }
                }
            }
        }

        stage('Manual Approval (Prod Gate)') {
            when { branch 'main' }
            steps {
                input message: 'Approve Production Deployment?'
            }
        }

        stage('Deploy to Production') {
            when { branch 'main' }
            steps {
                sh 'kubectl apply -f k8s/prod/'
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully for branch: ${env.BRANCH_NAME}"
        }
        failure {
            echo "❌ Pipeline failed for branch: ${env.BRANCH_NAME}"
        }
        always {
            sh 'docker system prune -f || true'
        }
    }
}
