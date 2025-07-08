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
                    echo "###### INITIALIZE STAGE - STARTED ######"
                    echo "Workspace path: ${WORKSPACE}"
                    
                    // 1. Verify workspace contents
                    sh """
                        echo "### WORKSPACE CONTENTS ###"
                        ls -al
                        echo "### PACKAGE.JSON ###"
                        cat package.json || echo "package.json not found"
                        echo "#########################"
                    """
                    
                    // 2. Test getMsName() independently
                    echo "Testing getMsName()..."
                    try {
                        env.MS_NAME = getMsName()
                        echo "MS_NAME successfully set to: ${env.MS_NAME}"
                    } catch (Exception e) {
                        echo "CRITICAL: getMsName() failed"
                        echo "Error details: ${e.toString()}"
                        echo "JOB_NAME value: ${env.JOB_NAME}"
                        error("Initialize failed at getMsName()")
                    }
                    
                    // 3. Test getTag() independently
                    echo "Testing getTag()..."
                    try {
                        env.IMAGE_TAG = getTag()
                        echo "IMAGE_TAG successfully set to: ${env.IMAGE_TAG}"
                    } catch (Exception e) {
                        echo "CRITICAL: getTag() failed"
                        echo "Error details: ${e.toString()}"
                        echo "package.json content:"
                        sh "cat package.json || echo 'package.json not accessible'"
                        error("Initialize failed at getTag()")
                    }
                    
                    echo "###### INITIALIZE STAGE - COMPLETED ######"
                }
            }
        }
        
        // Rest of your stages remain the same...
    }
}

def getMsName() {
    echo "DEBUG: Raw JOB_NAME = '${env.JOB_NAME}'"
    def parts = env.JOB_NAME.split("/")
    if (parts.size() == 0) {
        error "JOB_NAME appears empty or invalid"
    }
    def name = parts[0].toLowerCase()
    echo "DEBUG: Derived MS_NAME = '${name}'"
    return name
}

def getTag() {
    // Verify file exists and is readable
    if (!fileExists('package.json')) {
        error "package.json not found in workspace"
    }
    
    // Test JSON parsing
    def packageJson
    try {
        packageJson = readJSON file: 'package.json'
    } catch (Exception e) {
        error "Failed to parse package.json: ${e.getMessage()}"
    }
    
    // Verify version field exists
    if (!packageJson.version) {
        error "package.json is missing version field"
    }
    
    def version = packageJson.version
    def tag
    
    switch(env.BRANCH_NAME) {
        case 'main':
            tag = version
            break
        case 'develop':
            tag = "${version}-develop"
            break
        default:
            tag = "${version}-${env.BRANCH_NAME}"
    }
    
    echo "DEBUG: Version = ${version}, Branch = ${env.BRANCH_NAME}, Final Tag = ${tag}"
    return tag
}
