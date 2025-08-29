pipeline {
    agent any
     
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonarqube-scanner'
        DOCKER_REPO  = "zaid57/swiggy-clone"
    }
     
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/zainwaseem/a-swiggy-clone.git'
            }
        }

        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Swiggy-CI \
                        -Dsonar.projectKey=Swiggy-CI '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonarqube-token' 
                }
            } 
        }

        stage('Install Dependencies') {
            steps { sh "npm install" }
        }

        stage('TRIVY FS SCAN') {
            steps { sh "trivy fs . > trivyfs.txt" }
        }

        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {   
                        sh """
                          docker build -t ${DOCKER_REPO}:${BUILD_NUMBER} .
                          docker push ${DOCKER_REPO}:${BUILD_NUMBER}
                        """
                    }
                }
            }
        }

        stage("TRIVY Image Scan") {
            steps {
                sh "trivy image ${DOCKER_REPO}:${BUILD_NUMBER} > trivyimage.txt"
            }
        }

        stage('Trigger CD (GitOps)') {
            steps {
                script {
                    build job: 'swiggy-cd-pipeline', wait: false, parameters: [
                        string(name: 'IMAGE_REPO',  value: "${DOCKER_REPO}"),
                        string(name: 'IMAGE_TAG',   value: "${BUILD_NUMBER}"),
                        string(name: 'GITOPS_REPO', value: 'https://github.com/zainwaseem/gitops-register-app.git'),
                        string(name: 'DEPLOY_FILE', value: 'deployment.yml'),
                        string(name: 'BRANCH',      value: 'main')
                    ]
                }
            }
        }
    }
}
