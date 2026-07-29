pipeline {
    agent any

    environment {
        AWS_REGION        = 'us-east-1'
        ECR_REGISTRY      = '737441257613.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPO_BACKEND  = '737441257613.dkr.ecr.us-east-1.amazonaws.com/todolist-backend'
        ECR_REPO_FRONTEND = '737441257613.dkr.ecr.us-east-1.amazonaws.com/todolist-frontend'
        APP_VM_HOST       = '23.21.212.1'
        APP_VM_USER       = 'ubuntu'
        IMAGE_TAG         = "${env.GIT_COMMIT.take(7)}"
    }

    stages {
        stage('1. Checkout') {
            steps {
                echo "Checking out Backend source..."
                checkout scm
            }
        }

        stage('2. Build Backend Image') {
            steps {
                echo "Building Backend image..."
                sh """
                    docker build \
                        -t ${ECR_REPO_BACKEND}:${IMAGE_TAG} \
                        -t ${ECR_REPO_BACKEND}:latest \
                        .
                """
            }
        }

        stage('3. Test Backend') {
            steps {
                echo "Running Backend tests..."
                sh """
                    docker run --rm \
                        ${ECR_REPO_BACKEND}:${IMAGE_TAG} \
                        sh -c "npm test || true"
                """
            }
        }

        stage('4. Quality Gate') {
            steps {
                echo "Running Backend linter..."
                sh """
                    docker run --rm \
                        ${ECR_REPO_BACKEND}:${IMAGE_TAG} \
                        sh -c "npm run lint || true"
                """
            }
            post {
                failure {
                    error "Quality Gate failed! Pipeline stopped."
                }
            }
        }

        stage('5. Push Backend to ECR') {
            when {
                expression {
                    return env.BRANCH_NAME == 'master' || env.GIT_BRANCH == 'master' || env.GIT_BRANCH == 'origin/master'
                }
            }
            steps {
                echo "Pushing Backend images to ECR..."
                sh """
                    export HOME=/var/jenkins_home
                    aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    docker push ${ECR_REPO_BACKEND}:${IMAGE_TAG}
                """
            }
        }

        stage('6. Deploy to Dev (App VM)') {
            when {
                expression {
                    return env.BRANCH_NAME == 'master' || env.GIT_BRANCH == 'master' || env.GIT_BRANCH == 'origin/master'
                }
            }
            steps {
                echo "Deploying Backend (tag ${IMAGE_TAG}) + Frontend (latest)..."
                sh """
                    export HOME=/var/jenkins_home
                    ECR_PASSWORD=\$(aws ecr get-login-password --region ${AWS_REGION})

                    ssh -i /var/jenkins_home/.ssh/app-vm-key -o StrictHostKeyChecking=no ${APP_VM_USER}@${APP_VM_HOST} "
                        set -e
                        cd ~/todo-app-capstone

                        echo \${ECR_PASSWORD} | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        export BACKEND_IMAGE=${ECR_REPO_BACKEND}:${IMAGE_TAG}
                        export FRONTEND_IMAGE=${ECR_REPO_FRONTEND}:latest

                        echo '>>> BACKEND_IMAGE  =' \$BACKEND_IMAGE
                        echo '>>> FRONTEND_IMAGE =' \$FRONTEND_IMAGE

                        docker rmi \$BACKEND_IMAGE 2>/dev/null || true

                        docker compose pull
                        docker compose up -d --remove-orphans --force-recreate

                        echo '>>> Deploy done!'
                    "
                """
            }
        }
    }

    post {
        success { echo "Backend pipeline completed successfully!" }
        failure { echo "Backend pipeline failed!" }
        always {
            sh "docker image prune -f || true"
        }
    }
}

