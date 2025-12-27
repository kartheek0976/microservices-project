pipeline {
    agent any

    stages {

        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh 'docker build -t kartheek0976/adservice:latest .'
                    }
                }
            }
        }

        stage('Scan Docker Image with Trivy') {
            steps {
                sh '''
                  trivy image --severity HIGH,CRITICAL \
                  --no-progress \
                  kartheek0976/adservice:latest
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh 'docker push kartheek0976/adservice:latest'
                    }
                }
            }
        }
    }
}
