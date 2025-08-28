pipeline {
    agent {
        docker {
            image 'python:3.9-slim'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        ECR_REGISTRY = "992382545251.dkr.ecr.us-east-1.amazonaws.com"
        ECR_REPOSITORY = 'lir-repo'
        AWS_DEFAULT_REGION = "us-east-1"
        PROD_HOST = "10.0.1.83"
    }
    
    stages {
        stage('Setup') {
            steps {
                script {
                    // Install dependencies
                    sh '''
                        apt-get update
                        apt-get install -y curl docker.io awscli
                        pip install -r requirements.txt
                    '''
                }
            }
        }
        
        stage('Build Container Image') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        env.IMAGE_TAG = "latest-${BUILD_NUMBER}"
                    } else if (env.BRANCH_NAME.startsWith('PR-')) {
                        env.IMAGE_TAG = "pr-${CHANGE_ID}-${BUILD_NUMBER}"
                    } else {
                        env.IMAGE_TAG = "${BRANCH_NAME}-${BUILD_NUMBER}"
                    }
                    
                    sh """
                        docker build -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG} .
                    """
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    sh '''
                        # Run unit tests
                        python -m unittest discover -s tests -v
                        
                        # Create test results artifact
                        mkdir -p test-results
                        python -m unittest discover -s tests -v > test-results/test-output.txt 2>&1 || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'test-results/*.txt', fingerprint: true
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    sh """
                        # Login to ECR using IAM role
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        
                        # Push image
                        docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
                        
                        echo "Pushed image: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
                    """
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sshagent(['production-ssh-key']) {
                        sh """
                            # Deploy to production EC2
                            ssh -o StrictHostKeyChecking=no ec2-user@${PROD_HOST} '
                                # Login to ECR on production server using IAM role
                                aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                                
                                # Pull new image
                                docker pull ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
                                
                                # Stop existing container
                                docker stop calculator-app || true
                                docker rm calculator-app || true
                                
                                # Run new container
                                docker run -d --name calculator-app -p 5000:5000 ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
                                
                                # Wait for container to start
                                sleep 10
                            '
                        """
                    }
                }
            }
        }
        
        stage('Health Verification') {
            when {
                branch 'master'
            }
            steps {
                script {
                    // Health check with retries
                    sh """
                        for i in \$(seq 1 10); do
                            echo "Health check attempt \$i"
                            if curl -f http://${PROD_HOST}:5000/health; then
                                echo "Health check passed!"
                                exit 0
                            fi
                            sleep 5
                        done
                        echo "Health check failed after 10 attempts"
                        exit 1
                    """
                }
            }
        }
    }
    
    post {
        always {
            script {
                sh """
                    echo "Pipeline completed for branch: ${BRANCH_NAME}"
                    echo "Image built: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
                """
            }
        }
        failure {
            script {
                if (env.BRANCH_NAME == 'master') {
                    echo "Production deployment failed! Rolling back..."
                    // Add rollback logic here if needed
                }
            }
        }
    }
}
