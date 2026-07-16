// Ansible-driven CI/CD demo pipeline
// Flow: Checkout -> Test -> Docker Build -> Docker Push (DockerHub) -> Deploy (Ansible -> Kubernetes) -> Verify
//
// One-time setup before running:
//   1. Jenkins > Manage Jenkins > Credentials > add "Username with password"
//      with ID "DOCKERHUB_CREDENTIALS" (your DockerHub username + access token)
//   2. Update IMAGE_NAME below to <your-dockerhub-username>/ibm-ansible-demo
//   3. Run `kubectl apply -f k8s/` once manually so the Deployment/Service exist
//      before this pipeline tries to `kubectl set image` on it
//   4. Jenkins agent (native install) needs docker, kubectl, and node on PATH,
//      and its kubectl context must already point at docker-desktop
//   5. Install WSL2 (if not already present from Docker Desktop) and, inside
//      the WSL distro: `sudo apt update && sudo apt install -y ansible`.
//      Verify `kubectl get deployments` works from inside WSL too -- Docker
//      Desktop shares its docker-desktop context with WSL automatically.

pipeline {
    agent any

    environment {
        IMAGE_NAME = "vamandeshmukh/ibm-ansible-demo"
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/avianage/ibm-ansible-demo-4-jun-2026.git'
            }
        }

        stage('Install & Test') {
            steps {
                dir('app') {
                    bat 'npm install'
                    bat 'npm test'
                }
            }
        }

        stage('Docker Build') {
            steps {
                dir('app') {
                    bat "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_CREDENTIALS', usernameVariable: 'DOCKERHUB_CREDENTIALS_USR', passwordVariable: 'DOCKERHUB_CREDENTIALS_PSW')]) {
                    bat '''
                    @echo off
                    powershell -Command "$env:DOCKERHUB_CREDENTIALS_PSW | docker login -u $env:DOCKERHUB_CREDENTIALS_USR --password-stdin"
                    '''
                    bat "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    bat "docker push ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Deploy to Kubernetes (Ansible)') {
            steps {
                dir('ansible') {
                    // Ansible has no native Windows control node -- it runs inside
                    // WSL2, which Docker Desktop already relies on for most
                    // Windows setups. WSL2's kubectl shares the same
                    // docker-desktop context as Windows, so no extra kubeconfig
                    // setup is needed.
                    bat "wsl ansible-playbook -i inventory.ini deploy-playbook.yml --extra-vars \"image_tag=${IMAGE_TAG}\""
                }
            }
        }

        stage('Verify') {
            steps {
                bat "kubectl get pods -l app=ibm-ansible-demo"
                // Docker Desktop's Kubernetes exposes NodePort services on localhost directly
                bat "curl -s http://localhost:30081/health || true"
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded -- ${IMAGE_NAME}:${IMAGE_TAG} is live on Kubernetes"
        }
        failure {
            echo "Pipeline failed -- check the stage logs above"
        }
    }
}
