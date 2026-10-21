pipeline {
    agent any

    parameters {
        booleanParam(name: 'DEPLOY_FAULTY', defaultValue: false,
                     description: 'Tick to build the deliberately faulty version')
    }

    environment {
        IMAGE_NAME  = 'lakshmanan7777/placement-portal'
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
        DOCKER_CRED = 'dockerhub-creds'   // Username with password
        KUBE_CRED   = 'kubeconfig'        // Secret file
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main',
    url: 'https://github.com/lakshmananram7777-dev/placement-portal-.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def dockerfile = params.DEPLOY_FAULTY ? 'faulty/Dockerfile' : 'Dockerfile'
                    bat "docker build -f ${dockerfile} -t %IMAGE_NAME%:%IMAGE_TAG% -t %IMAGE_NAME%:latest ."
                }
            }
        }

        stage('Test Image') {
            steps {
                bat 'docker image inspect %IMAGE_NAME%:%IMAGE_TAG%'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_CRED,
                        usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    bat 'echo %DH_PASS%| docker login -u %DH_USER% --password-stdin'
                    bat 'docker push %IMAGE_NAME%:%IMAGE_TAG%'
                    bat 'docker push %IMAGE_NAME%:latest'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: env.KUBE_CRED, variableId: 'KUBECONFIG_FILE')]) {
                    bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl apply -f deployment.yaml'
                    bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl set image deployment/placement-portal placement-portal=%IMAGE_NAME%:%IMAGE_TAG%'
                }
            }
        }

        stage('Verify Rollout') {
            steps {
                withCredentials([file(credentialsId: env.KUBE_CRED, variableId: 'KUBECONFIG_FILE')]) {
                    bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl rollout status deployment/placement-portal --timeout=60s'
                    bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl get pods'
                }
            }
        }
    }

    post {
        failure {
            echo 'Rollout failed. Rolling back to the previous stable release...'
            withCredentials([file(credentialsId: env.KUBE_CRED, variableId: 'KUBECONFIG_FILE')]) {
                bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl rollout undo deployment/placement-portal'
                bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl rollout status deployment/placement-portal --timeout=60s'
                bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl get pods'
            }
        }
        success {
            echo 'Deployment successful.'
        }
    }
}
