pipeline {

    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'

        AWS_REGION = 'us-east-1'
        EKS_CLUSTER_NAME = 'kastro-eks'

        ECR_REPOSITORY = '685459860804.dkr.ecr.us-east-1.amazonaws.com/bookmyshow'
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

                sh 'ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=BMS \
                            -Dsonar.projectKey=BMS
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate(
                        abortPipeline: false,
                        credentialsId: 'Sonar-token'
                    )
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    cd bookmyshow-app

                    if [ -f package.json ]; then
                        npm install
                    else
                        echo "Error: package.json not found!"
                        exit 1
                    fi
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
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    echo "Logging in to Amazon ECR..."

                    aws ecr get-login-password \
                    --region $AWS_REGION | \
                    docker login \
                    --username AWS \
                    --password-stdin \
                    685459860804.dkr.ecr.us-east-1.amazonaws.com
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."

                    docker build \
                    -t $ECR_REPOSITORY:$BUILD_NUMBER \
                    -f bookmyshow-app/Dockerfile \
                    bookmyshow-app
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    echo "Pushing Docker image to ECR..."

                    docker push \
                    $ECR_REPOSITORY:$BUILD_NUMBER
                '''
            }
        }

        stage('Ansible Kubernetes Deployment') {
            steps {
                sh '''
                    echo "Running Ansible deployment..."

                    ansible-playbook \
                    -i ansible/inventory \
                    ansible/deploy.yml \
                    -e image_tag=$BUILD_NUMBER
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Verifying Kubernetes deployment..."

                    kubectl get deployment bms-app

                    echo "Checking pods..."

                    kubectl get pods -o wide

                    echo "Checking LoadBalancer..."

                    kubectl get svc bms-service
                '''
            }
        }
    }

    post {

        always {
            emailext(
                attachLog: true,
                subject: "${currentBuild.result}",
                body:
                    "Project: ${env.JOB_NAME}<br/>" +
                    "Build Number: ${env.BUILD_NUMBER}<br/>" +
                    "URL: ${env.BUILD_URL}<br/>",
                to: 'kastrokiran@gmail.com',
                attachmentsPattern: 'trivyfs.txt'
            )
        }

        success {
            echo 'Book-My-Show CI/CD pipeline completed successfully!'
        }
    }
}
