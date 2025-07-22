def registry= "654654385216.dkr.ecr.us-east-1.amazonaws.com"
def region = "us-east-1"

pipeline {
    agent any
    
    environment {
        TAG = ""
        MS = "result-image"
    }
    stages{
        stage("init"){
            steps{
                script{
                    TAG = getTag()
                    echo "Image tag set to: ${TAG}
                
                }
            }
        }
        stage("Build Docker image"){
            steps{
                script{
                    sh "docker build . -t ${registry}/${env.MS}:${env.TAG}"
                }
            }
        }

        stage("Login to Ecr"){
            steps{
                script{
                    withAWS(region:"$region",credentials:'aws_creds'){
                        sh "aws ecr get-login-password --region ${region} | docker login --username AWS --password-stdin ${registry}"
                    }
                }
            }
        }

        stage("Docker push"){
            steps{
                script{
                    withAWS(region:"$region",credentials:'aws_creds'){
                        sh "docker push ${registry}/${env.MS}:${env.TAG}"
                    }
                }
            }
        }

        stage("Deploy to Dev"){
            when{branch 'develop'}
            steps{
                script{
                    withAWS(region:"$region",credentials:'aws_creds'){
                        sh "aws eks update-kubeconfig --name vote-dev"
                        sh 'curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.5/2024-01-04/bin/linux/amd64/kubectl'  
                        sh 'chmod u+x ./kubectl'
                        sh "./kubectl get deployment ${env.MS} -n vote || ./kubectl apply -f k8s/deployment.yaml -n vote"
                        sh "./kubectl set image deploy/result result=${registry}/result-image:${tag} -n vote"
                        sh "./kubectl rollout restart deploy/${env.MS} -n vote"
                    }
                }
            }
        }
    }
}

def getMsName(){
    print env.JOB_NAME
    return env.JOB_NAME.split("/")[0]
}

def getTag(){
 sh "ls -l"
 version = readJSON file: 'package.json'
 version = version["version"]
 print "version: ${version}"

 def tag = ""
  if (env.BRANCH_NAME == "main"){
    tag = version
  } else if(env.BRANCH_NAME == "develop"){
    tag = "${version}-develop"
  } else {
    tag = "${version}-${env.BRANCH_NAME}"
  }
return tag 
}
