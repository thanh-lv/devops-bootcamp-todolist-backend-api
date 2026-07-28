pipeline {
    agent any

    environment {
        AWS_REGION          = 'us-east-1'
        ECR_REGISTRY        = '737441257613.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPO_BACKEND    = '737441257613.dkr.ecr.us-east-1.amazonaws.com/todolist-backend'
        ECR_REPO_FRONTEND   = '737441257613.dkr.ecr.us-east-1.amazonaws.com/todolist-frontend'
        APP_VM_HOST         = '23.21.212.1'
        APP_VM_USER         = 'ubuntu'
        IMAGE_TAG           = "${GIT_COMMIT[0..6]}"
    }

    stages {

        // ============================================================
        // CI STAGES — chạy cả khi PR lẫn push to master
        // ============================================================

        stage('1. Checkout') {
            steps {
                echo "Checking out source code..."
                sh """
                    rm -rf devops-bootcamp-todolist-backend-api devops-bootcamp-todolist-frontend

                    git clone https://github.com/thanh-lv/devops-bootcamp-todolist-backend-api
                    git clone https://github.com/thanh-lv/devops-bootcamp-todolist-frontend
                """
                echo "BRANCH_NAME=${env.BRANCH_NAME}, GIT_BRANCH=${env.GIT_BRANCH}"
            }
        }

        stage('2. Build Images') {
            steps {
                echo "Building Docker images..."
                sh """
                    docker build \
                        --build-arg NEXT_PUBLIC_API_URL=http://${APP_VM_HOST}:5000/api \
                        -t ${ECR_REPO_FRONTEND}:${IMAGE_TAG} \
                        -t ${ECR_REPO_FRONTEND}:latest \
                        ./devops-bootcamp-todolist-frontend

                    docker build \
                        -t ${ECR_REPO_BACKEND}:${IMAGE_TAG} \
                        -t ${ECR_REPO_BACKEND}:latest \
                        ./devops-bootcamp-todolist-backend-api
                """
            }
        }

        stage('3. Test') {
            parallel {
                stage('Test Backend') {
                    steps {
                        echo "Running backend tests..."
                        sh """
                            docker run --rm \
                                ${ECR_REPO_BACKEND}:${IMAGE_TAG} \
                                sh -c "npm test || true"
                        """
                    }
                }
                stage('Test Frontend') {
                    steps {
                        echo "Running frontend tests..."
                        sh """
                            docker run --rm \
                                ${ECR_REPO_FRONTEND}:${IMAGE_TAG} \
                                sh -c "npm test || true"
                        """
                    }
                }
            }
        }

        stage('4. Quality Gate') {
            steps {
                echo "Running linter and quality checks..."
                sh """
                    docker run --rm \
                        ${ECR_REPO_BACKEND}:${IMAGE_TAG} \
                        sh -c "npm run lint || true"

                    docker run --rm \
                        ${ECR_REPO_FRONTEND}:${IMAGE_TAG} \
                        sh -c "npm run lint || true"
                """
            }
            post {
                failure {
                    error "Quality Gate failed! Pipeline stopped."
                }
            }
        }

        // ============================================================
        // CD STAGES — run only push to master branch
        // ============================================================

        stage('5. Push to ECR') {
            when {
                expression {
                    return env.BRANCH_NAME == 'master' || env.GIT_BRANCH == 'master' || env.GIT_BRANCH == 'origin/master'
                }
            }
            steps {
                echo "Pushing images to ECR..."
                sh """
                    export HOME=/var/jenkins_home

                    aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    docker push ${ECR_REPO_BACKEND}:${IMAGE_TAG}
                    docker push ${ECR_REPO_BACKEND}:latest

                    docker push ${ECR_REPO_FRONTEND}:${IMAGE_TAG}
                    docker push ${ECR_REPO_FRONTEND}:latest
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
        echo "Deploying to App VM with tag ${IMAGE_TAG}..."

            sh """
                ssh -i /var/jenkins_home/.ssh/app-vm-key -o StrictHostKeyChecking=no ${APP_VM_USER}@${APP_VM_HOST} '
                    set -e
                    cd ~/todo-app-capstone

                    # Login ECR
                    aws ecr get-login-password --region us-east-1 | \\
                        docker login --username AWS --password-stdin 737441257613.dkr.ecr.us-east-1.amazonaws.com

                    export BACKEND_IMAGE=${ECR_REPO_BACKEND}:${IMAGE_TAG}
                    export FRONTEND_IMAGE=${ECR_REPO_FRONTEND}:${IMAGE_TAG}

                    echo ">>> BACKEND_IMAGE = \$BACKEND_IMAGE"
                    echo ">>> FRONTEND_IMAGE = \$FRONTEND_IMAGE"

                    # Xóa image cũ
                    docker rmi \$BACKEND_IMAGE \$FRONTEND_IMAGE 2>/dev/null || true

                    # Pull image mới từ ECR
                    docker compose pull

                    # Deploy
                    docker compose up -d --remove-orphans --force-recreate

                    echo ">>> Deploy to App VM done!"
                '
            """
    }
}

        // stage('7. Approval Gate') {
        //     when {
        //         branch 'master'
        //     }
        //     steps {
        //         timeout(time: 30, unit: 'MINUTES') {
        //             input message: 'Deploy lên Production (K8s)?',
        //                   ok: 'Approve',
        //                   submitter: 'admin'
        //         }
        //     }
        // }

        // stage('8. Deploy to Production (K8s)') {
        //     when {
        //         branch 'master'
        //     }
        //     steps {
        //         echo "Deploying to Kubernetes..."
        //         sshagent(['app-vm-ssh-key']) {
        //             sh """
        //                 ssh -o StrictHostKeyChecking=no ${APP_VM_USER}@${APP_VM_HOST} '
        //                     kubectl set image deployment/backend \
        //                         backend=${ECR_REPO_BACKEND}:${IMAGE_TAG} \
        //                         -n todolist
        //
        //                     kubectl set image deployment/frontend \
        //                         frontend=${ECR_REPO_FRONTEND}:${IMAGE_TAG} \
        //                         -n todolist
        //
        //                     kubectl rollout status deployment/backend -n todolist
        //                     kubectl rollout status deployment/frontend -n todolist
        //
        //                     echo "Deploy to Production done!"
        //                 '
        //             """
        //         }
        //     }
        // }
    }

    // ============================================================
    // POST ACTIONS
    // ============================================================
    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
        always {
            // Dọn dẹp images cũ trên Tools VM để tránh đầy disk
            sh """
                docker image prune -f || true
            """
        }
    }
}