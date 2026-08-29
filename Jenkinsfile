pipeline {

    agent any

    environment {

        // AWS
        AWS_REGION = 'us-east-1'

        // ECR
        ECR_REGISTRY = '685459860804.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY = '685459860804.dkr.ecr.us-east-1.amazonaws.com/bookmyshow'

        // Docker
        IMAGE_NAME = 'bookmyshow'

        // Kubernetes
        EKS_CLUSTER = 'kastro-eks'
        DEPLOYMENT_NAME = 'bms-app'
        CONTAINER_NAME = 'bms-container'
        SERVICE_NAME = 'bms-service'

        // Application
        APP_DIR = 'bookmyshow-app'
    }

    stages {

        /*
         * ============================================================
         * CLEAN WORKSPACE
         * ============================================================
         */
        stage('Clean Workspace') {
            steps {
                echo '========== CLEANING WORKSPACE =========='
                cleanWs()
            }
        }


        /*
         * ============================================================
         * CHECKOUT CODE
         * ============================================================
         */
        stage('Checkout from Git') {
            steps {
                echo '========== CHECKING OUT CODE FROM GITHUB =========='

                git branch: 'main',
                    url: 'https://github.com/deeprajsingh14/Book-My-Show.git'

                echo 'Git checkout completed successfully'

                sh '''
                    echo "========== PROJECT FILES =========="
                    ls -la

                    echo "========== ANSIBLE FILES =========="
                    ls -la ansible

                    echo "========== APPLICATION FILES =========="
                    ls -la bookmyshow-app
                '''
            }
        }


        /*
         * ============================================================
         * INSTALL NODE DEPENDENCIES
         * ============================================================
         */
        stage('Install Dependencies') {
            steps {
                echo '========== INSTALLING NODE DEPENDENCIES =========='

                sh '''
                    node -v
                    npm -v

                    cd ${APP_DIR}

                    if [ -f package.json ]; then
                        npm install
                    else
                        echo "ERROR: package.json not found"
                        exit 1
                    fi

                    echo "Dependencies installed successfully"
                '''
            }
        }


        /*
         * ============================================================
         * OWASP FILESYSTEM SCAN
         * ============================================================
         *
         * No Jenkins dependencyCheck DSL is used here.
         * This prevents:
         * No such DSL method 'dependencyCheck'
         *
         */
        stage('OWASP FS Scan') {
            steps {
                echo '========== OWASP FILESYSTEM SCAN =========='

                sh '''
                    if command -v dependency-check.sh >/dev/null 2>&1; then

                        echo "Dependency-Check found"

                        dependency-check.sh \
                            --scan . \
                            --format HTML \
                            --out dependency-check-report

                    else

                        echo "OWASP Dependency-Check is not installed."
                        echo "Skipping OWASP Dependency-Check scan."

                    fi
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'dependency-check-report/**',
                        allowEmptyArchive: true
                }
            }
        }


        /*
         * ============================================================
         * TRIVY FILESYSTEM SCAN
         * ============================================================
         */
        stage('Trivy FS Scan') {
            steps {
                echo '========== TRIVY FILESYSTEM SCAN =========='

                sh '''
                    if command -v trivy >/dev/null 2>&1; then

                        trivy fs . > trivyfs.txt || true

                        echo "Trivy filesystem scan completed"

                    else

                        echo "ERROR: Trivy is not installed"
                        exit 1

                    fi
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'trivyfs.txt',
                        allowEmptyArchive: true
                }
            }
        }


        /*
         * ============================================================
         * LOGIN TO AWS ECR
         * ============================================================
         */
        stage('Login to ECR') {
            steps {
                echo '========== LOGIN TO AWS ECR =========='

                sh '''
                    aws --version

                    aws ecr get-login-password \
                        --region ${AWS_REGION} | \
                    docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}

                    echo "Successfully logged in to ECR"
                '''
            }
        }


        /*
         * ============================================================
         * BUILD DOCKER IMAGE
         * ============================================================
         */
        stage('Build Docker Image') {
            steps {
                echo '========== BUILDING DOCKER IMAGE =========='

                sh '''
                    docker --version

                    cd ${APP_DIR}

                    docker build \
                        -t ${IMAGE_NAME}:${BUILD_NUMBER} .

                    echo "Docker image built successfully"

                    docker images | grep ${IMAGE_NAME}
                '''
            }
        }


        /*
         * ============================================================
         * TRIVY DOCKER IMAGE SCAN
         * ============================================================
         */
        stage('Trivy Image Scan') {
            steps {
                echo '========== TRIVY DOCKER IMAGE SCAN =========='

                sh '''
                    trivy image \
                        ${IMAGE_NAME}:${BUILD_NUMBER} \
                        > trivy-image.txt || true

                    echo "Trivy image scan completed"
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'trivy-image.txt',
                        allowEmptyArchive: true
                }
            }
        }


        /*
         * ============================================================
         * TAG DOCKER IMAGE
         * ============================================================
         */
        stage('Tag Docker Image') {
            steps {
                echo '========== TAGGING DOCKER IMAGE =========='

                sh '''
                    docker tag \
                        ${IMAGE_NAME}:${BUILD_NUMBER} \
                        ${ECR_REPOSITORY}:${BUILD_NUMBER}

                    echo "Image tagged successfully"

                    docker images | grep bookmyshow
                '''
            }
        }


        /*
         * ============================================================
         * PUSH IMAGE TO ECR
         * ============================================================
         */
        stage('Push Docker Image to ECR') {
            steps {
                echo '========== PUSHING IMAGE TO AWS ECR =========='

                sh '''
                    docker push \
                        ${ECR_REPOSITORY}:${BUILD_NUMBER}

                    echo "Docker image pushed successfully"

                    echo "Image:"
                    echo "${ECR_REPOSITORY}:${BUILD_NUMBER}"
                '''
            }
        }


        /*
         * ============================================================
         * ANSIBLE KUBERNETES DEPLOYMENT
         * ============================================================
         */
        stage('Ansible Kubernetes Deployment') {
            steps {
                echo '========== DEPLOYING TO EKS USING ANSIBLE =========='

                sh '''
                    ansible --version
                    kubectl version --client

                    echo "========== RUNNING ANSIBLE PLAYBOOK =========="

                    ansible-playbook \
                        -i ansible/inventory \
                        ansible/deploy.yml \
                        -e image_tag=${BUILD_NUMBER}

                    echo "Ansible Kubernetes deployment completed"
                '''
            }
        }


        /*
         * ============================================================
         * VERIFY KUBERNETES DEPLOYMENT
         * ============================================================
         */
        stage('Verify Deployment') {
            steps {
                echo '========== VERIFYING KUBERNETES DEPLOYMENT =========='

                sh '''
                    echo "========== EKS CLUSTER =========="

                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${EKS_CLUSTER}

                    echo "========== DEPLOYMENT =========="

                    kubectl get deployment ${DEPLOYMENT_NAME}

                    echo "========== PODS =========="

                    kubectl get pods -o wide

                    echo "========== SERVICE =========="

                    kubectl get svc ${SERVICE_NAME}

                    echo "========== LOAD BALANCER =========="

                    kubectl get svc ${SERVICE_NAME} \
                        -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

                    echo ""
                '''
            }
        }

    }


    /*
     * ================================================================
     * POST ACTIONS
     * ================================================================
     */
    post {

        success {
            echo '''
            ==========================================
                 BOOK-MY-SHOW DEPLOYMENT SUCCESS
            ==========================================
            '''

            echo "Build Number: ${BUILD_NUMBER}"
            echo "Docker Image: ${ECR_REPOSITORY}:${BUILD_NUMBER}"
            echo "EKS Cluster: ${EKS_CLUSTER}"
        }

        failure {
            echo '''
            ==========================================
                 BOOK-MY-SHOW DEPLOYMENT FAILED
            ==========================================
            '''

            echo "Build Number: ${BUILD_NUMBER}"
            echo "Please check the Jenkins console output."
        }

        always {
            echo '========== PIPELINE COMPLETED =========='
        }
    }
}[201~
