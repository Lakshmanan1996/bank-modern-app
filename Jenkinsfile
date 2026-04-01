pipeline {
    agent none

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        DOCKERHUB_USER = "lakshvar96"
        IMAGE = "banking"
        
        GIT_REPO = "https://github.com/Lakshmanan1996/bank-modern-app.git"
    }

    stages {

        /* ===================== CHECKOUT ===================== */
        stage('Checkout Code') {
            agent { label 'workernode1' }
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: 'main']],
                    userRemoteConfigs: [[url: "${GIT_REPO}"]]
                ])
            }
        }

        /* ===================== STASH SOURCE ===================== */
        stage('Stash Source') {
            agent { label 'workernode1' }
            steps {
                stash includes: '**/*', name: 'source-code'
            }
        }

        /* ===================== SONARQUBE ===================== */
        stage('SonarQube Analysis') {
            agent { label 'workernode2' }
            steps {
                unstash 'source-code'

                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('sonarqube') {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                          -Dsonar.projectKey=banking \
                          -Dsonar.projectName=banking \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=**/node_modules/**
                        """
                    }
                }
            }
        }

        /*===================== QUALITY GATE ===================== */
        
        stage('Quality Gate') {
            agent { label 'workernode2' }
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
      

        /* ===================== DOCKER BUILD ===================== */
        stage('Docker Build') {
            agent { label 'workernode3' }
            steps {
                unstash 'source-code'

                sh """
                # Frontend Build
                docker build -t ${DOCKERHUB_USER}/${IMAGE}:${BUILD_NUMBER} 
                docker tag ${DOCKERHUB_USER}/${IMAGE}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE}:latest

                
            }
        }

        /* ===================== TRIVY SCAN ===================== */
        stage('Trivy Scan') {
            agent { label 'workernode3' }
            steps {
                sh """
                trivy image --exit-code 0 --severity HIGH, CRITICAL ${DOCKERHUB_USER}/${IMAGE}:${BUILD_NUMBER}
                """
            }
        }

        /* ===================== PUSH TO DOCKER HUB ===================== */
        stage('Push Image') {
            agent { label 'workernode3' }
            steps {
                unstash 'source-code'

                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                }

                sh """
                docker push ${DOCKERHUB_USER}/${IMAGE}:${BUILD_NUMBER}
                docker push ${DOCKERHUB_USER}/${IMAGE}:latest

                """
            }
        }
    }

    post {
        success {
            echo "✅ banking CI Pipeline SUCCESS"
        }
        failure {
            echo "❌ banking CI Pipeline FAILED"
        }
    }
}
