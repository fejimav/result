def registry = "817041139384.dkr.ecr.us-east-1.amazonaws.com"
def region = "us-east-1"

pipeline {
    agent any
    environment {
        // Environment variables available in all stages
        BRANCH_NAME = "${env.BRANCH_NAME}"
    }
    stages {
        stage("Initialize") {
            steps {
                script {
                    // Properly declare variables with def
                    def ms = getMsName()
                    def tag = getTag()
                    
                    // Store in environment for other stages
                    env.MS_NAME = ms
                    env.IMAGE_TAG = tag
                    
                    echo "Building ${env.MS_NAME}:${env.IMAGE_TAG}"
                    echo "Branch: ${env.BRANCH_NAME}"
                    sh "ls -l"  // Debug file listing
                }
            }
        }
        
        stage("Build Docker Image") {
            steps {
                script {
                    sh "docker build . -t ${registry}/${env.MS_NAME}:${env.IMAGE_TAG}"
                }
            }
        }

        stage("Login to ECR") {
            steps {
                script {
                    withAWS(region: region, credentials: 'aws-ecr-creds') {
                        sh """
                            aws ecr get-login-password --region ${region} | \
                            docker login --username AWS --password-stdin ${registry}
                        """
                    }
                }
            }
        }

        stage("Docker Push") {
            steps {
                script {
                    withAWS(region: region, credentials: 'aws-ecr-creds') {
                        sh "docker push ${registry}/${env.MS_NAME}:${env.IMAGE_TAG}"
                    }
                }
            }
        }

        stage("Deploy to Dev") {
            when { 
                branch 'develop' 
            }
            steps {
                script {
                    withAWS(region: region, credentials: 'aws-ecr-creds') {
                        // Configure kubectl
                        sh "aws eks update-kubeconfig --name vote-dev --region ${region}"
                        
                        // Get kubectl
                        sh 'curl -LO https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.5/2024-01-04/bin/linux/amd64/kubectl'
                        sh 'chmod +x ./kubectl'
                        
                        // Deploy
                        sh """
                            ./kubectl get deployment result -n vote || \
                            ./kubectl apply -f k8s/deployment.yaml -n vote
                            
                            ./kubectl set image deploy/result \
                            result=${registry}/${env.MS_NAME}:${env.IMAGE_TAG} -n vote
                            
                            ./kubectl rollout restart deploy/result -n vote
                            ./kubectl rollout status deploy/result -n vote --timeout=300s
                        """
                    }
                }
            }
        }
    }
}

def getMsName() {
    return env.JOB_NAME.split("/")[0]
}

def getTag() {
    def version = readJSON file: 'package.json'
    version = version["version"]
    
    if (env.BRANCH_NAME == "main") {
        return version
    } else if (env.BRANCH_NAME == "develop") {
        return "${version}-develop"
    } else {
        return "${version}-${env.BRANCH_NAME}"
    }
}
