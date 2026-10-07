# Sonar-test
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'myapp:latest'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${IMAGE_NAME} .'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --ignorefile .trivyignore \
                      ${IMAGE_NAME}
                '''
            }
        }

        stage('Trivy Config Scan') {
            steps {
                sh '''
                    trivy config .
                '''
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar',
                                 fingerprint: true
            }
        }

        stage('Archive Trivy Report') {
            steps {
                sh '''
                    trivy image \
                      --format json \
                      --output trivy-report.json \
                      --ignorefile .trivyignore \
                      ${IMAGE_NAME}
                '''
                
                archiveArtifacts artifacts: 'trivy-report.json',
                                 fingerprint: true
            }
        }
    }
}
