
pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'DEPLOY_FAULTY',
            defaultValue: false,
            description: 'Tick to test automatic rollback'
        )
    }

    environment {
        IMAGE_NAME = 'lakshmanan7777/placement-portal'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_CRED = 'dockerhub-creds'
        KUBE_CRED = 'kubeconfig'
        DEPLOYMENT_STARTED = 'false'
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
                    def dockerfile = params.DEPLOY_FAULTY
                        ? 'faulty/Dockerfile'
                        : 'Dockerfile'

                    bat "docker build -f ${dockerfile} -t %IMAGE_NAME%:%IMAGE_TAG% ."
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
                withCredentials([
                    usernamePassword(
                        credentialsId: env.DOCKER_CRED,
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASS'
                    )
                ]) {
                    bat 'echo %DH_PASS%| docker login -u %DH_USER% --password-stdin'
                    bat 'docker push %IMAGE_NAME%:%IMAGE_TAG%'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: env.KUBE_CRED, variable: 'KUBECONFIG_FILE')
                ]) {
                    bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl apply -f deployment.yaml'

                    script {
                        env.DEPLOYMENT_STARTED = 'true'
                    }

                    bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl set image deployment/placement-portal placement-portal=%IMAGE_NAME%:%IMAGE_TAG%'
                }
            }
        }

        stage('Verify Rollout') {
            steps {
                withCredentials([
                    file(credentialsId: env.KUBE_CRED, variable: 'KUBECONFIG_FILE')
                ]) {
                    bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl rollout status deployment/placement-portal --timeout=60s'
                    bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl get pods'
                }
            }
        }
    }

    post {
        failure {
            script {
                if (env.DEPLOYMENT_STARTED == 'true') {
                    echo 'Deployment failed. Rolling back to the previous release.'

                    withCredentials([
                        file(credentialsId: env.KUBE_CRED, variable: 'KUBECONFIG_FILE')
                    ]) {
                        bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl rollout undo deployment/placement-portal'
                        bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl rollout status deployment/placement-portal --timeout=60s'
                        bat 'set KUBECONFIG=%KUBECONFIG_FILE% && kubectl get pods'
                    }
                } else {
                    echo 'Failure before deployment. No rollback required.'
                }
            }
        }

        success {
            echo 'Deployment successful.'
        }
    }
}
