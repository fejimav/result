def registry = "817041139384.dkr.ecr.us-east-1.amazonaws.com"
def region = "us-east-1"

pipeline {
    agent any
    options {
        timeout(time: 30, unit: 'MINUTES')
    }
    environment {
        BRANCH_NAME = "${env.BRANCH_NAME}"
    }
    stages {
        stage('Initialize') {
            steps {
                script {
                    try {
                        // Get and store microservice name and tag
                        env.MS_NAME = getMsName()
                        env.IMAGE_TAG = getTag()
                        
                        echo "=========================================="
                        echo "Build Parameters:"
                        echo "Microservice: ${env.MS_NAME}"
                        echo "Tag: ${env.IMAGE_TAG}"
                        echo "Branch: ${env.BRANCH_NAME}"
                        echo "Registry: ${registry}"
                        echo "=========================================="
                        
                        // Debug workspace contents
                        sh """
                            echo "Workspace contents:"
                            ls -al
                            echo "Package.json version:"
                            cat package.json | grep version
                        """
                    } catch (Exception e) {
                        error "Initialization failed: ${e.toString()}"
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    try {
                        sh """
                            docker build -t ${registry}/${env.MS_NAME}:${env.IMAGE_TAG} .
                            docker images | grep ${env.MS_NAME}
                        """
                    } catch (Exception e) {
                        error "Docker build failed: ${e.toString()}"
                    }
                }
            }
        }

        stage('Login to ECR') {
            steps {
                script {
                    withAWS(credentials: 'aws-ecr-creds', region: region) {
                        try {
                            sh """
                                aws ecr get-login-password --region ${region} | \
                                docker login --username AWS --password-stdin ${registry}
                            """
                        } catch (Exception e) {
                            error "ECR login failed: ${e.toString()}"
                        }
                    }
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withAWS(credentials: 'aws-ecr-creds', region: region) {
                        try {
                            sh """
                                docker push ${registry}/${env.MS_NAME}:${env.IMAGE_TAG}
                            """
                        } catch (Exception e) {
                            error "Docker push failed: ${e.toString()}"
                        }
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when { 
                branch 'develop' 
            }
            steps {
                script {
                    withAWS(credentials: 'aws-ecr-creds', region: region) {
                        try {
                            sh """
                                # Configure kubectl
                                aws eks update-kubeconfig --name vote-dev --region ${region}
                                
                                # Get kubectl
                                curl -LO https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.5/2024-01-04/bin/linux/amd64/kubectl
                                chmod +x ./kubectl
                                
                                # Deploy
                                ./kubectl get deployment result -n vote || ./kubectl apply -f k8s/deployment.yaml -n vote
                                ./kubectl set image deploy/result result=${registry}/${env.MS_NAME}:${env.IMAGE_TAG} -n vote
                                ./kubectl rollout restart deploy/result -n vote
                                ./kubectl rollout status deploy/result -n vote --timeout=300s
                                
                                # Verify deployment
                                ./kubectl get pods -n vote
                            """
                        } catch (Exception e) {
                            error "Deployment failed: ${e.toString()}"
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            echo "Pipeline completed - cleanup can go here"
            cleanWs()
        }
        failure {
            echo "Pipeline failed - sending notifications"
            // Add notification logic here (Slack, email, etc.)
        }
        success {
            echo "Pipeline succeeded!"
        }
    }
}

def getMsName() {
    try {
        return env.JOB_NAME.split("/")[0].toLowerCase()
    } catch (Exception e) {
        error "Failed to get microservice name: ${e.toString()}"
    }
}

def getTag() {
    try {
        def version = readJSON file: 'package.json'
        version = version["version"]
        
        if (env.BRANCH_NAME == "main") {
            return version
        } else if (env.BRANCH_NAME == "develop") {
            return "${version}-develop"
        } else {
            return "${version}-${env.BRANCH_NAME}"
        }
    } catch (Exception e) {
        error "Failed to get version tag: ${e.toString()}"
    }
}
