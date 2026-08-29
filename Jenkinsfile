pipeline {

    agent any

    environment {
        AWS_REGION       = 'us-east-1'
        EKS_CLUSTER_NAME = 'kastro-eks'

        AWS_ACCOUNT_ID   = '685459860804'
        ECR_REPOSITORY   = 'bookmyshow'
        ECR_REGISTRY     = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_TAG        = "${BUILD_NUMBER}"
        ECR_IMAGE        = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/deeprajsingh14/Book-My-Show.git'

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

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "========== INSTALLING NODE DEPENDENCIES =========="

                    cd bookmyshow-app

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

        stage('OWASP FS Scan') {
            steps {
                dependencyCheck(
                    additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
                    odcInstallation: 'DP-Check'
                )

                dependencyCheckPublisher(
                    pattern: '**/dependency-check-report.xml'
                )
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''
                    echo "========== TRIVY FILESYSTEM SCAN =========="

                    trivy fs . > trivyfs.txt || true

                    echo "Trivy filesystem scan completed"
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    echo "========== LOGIN TO AWS ECR =========="

                    aws sts get-caller-identity

                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login \
                    --username AWS \
                    --password-stdin ${ECR_REGISTRY}

                    echo "ECR login successful"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "========== BUILDING DOCKER IMAGE =========="

                    docker build \
                        --no-cache \
                        -t ${ECR_IMAGE} \
                        -f bookmyshow-app/Dockerfile \
                        bookmyshow-app

                    echo "Docker image created:"
                    docker images | grep bookmyshow
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    echo "========== TRIVY DOCKER IMAGE SCAN =========="

                    trivy image ${ECR_IMAGE} > trivyimage.txt || true

                    echo "Trivy image scan completed"
                '''
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                sh '''
                    echo "========== PUSHING IMAGE TO ECR =========="

                    docker push ${ECR_IMAGE}

                    echo "Image pushed successfully:"
                    echo "${ECR_IMAGE}"
                '''
            }
        }

        stage('Ansible Kubernetes Deployment') {
            steps {
                sh '''
                    echo "========== ANSIBLE EKS DEPLOYMENT =========="

                    ansible-playbook \
                        -i ansible/inventory \
                        ansible/deploy.yml \
                        -e image_tag=${IMAGE_TAG}

                    echo "Ansible deployment completed successfully"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "========== VERIFYING KUBERNETES DEPLOYMENT =========="

                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${EKS_CLUSTER_NAME}

                    echo ""
                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment bms-app

                    echo ""
                    echo "========== PODS =========="
                    kubectl get pods -o wide

                    echo ""
                    echo "========== SERVICE =========="
                    kubectl get svc bms-service

                    echo ""
                    echo "========== LOAD BALANCER URL =========="
                    kubectl get svc bms-service \
                        -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

                    echo ""
                    echo "======================================"
                '''
            }
        }
    }

    post {

        success {
            echo '''
            ==========================================
                 BOOK-MY-SHOW DEPLOYMENT SUCCESSFUL
            ==========================================
            '''
        }

        failure {
            echo '''
            ==========================================
                 BOOK-MY-SHOW DEPLOYMENT FAILED
            ==========================================
            '''
        }

        always {
            archiveArtifacts(
                artifacts: 'trivyfs.txt,trivyimage.txt',
                allowEmptyArchive: true
            )

            echo "Build Number: ${BUILD_NUMBER}"
            echo "Build Result: ${currentBuild.result}"
        }
    }
}
