pipeline {
    agent any
    tools {
        nodejs 'NodeJS-18'
    }
    environment {
        DOCKER_REPO = 'landmark-web-app'
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER = 'landmark-eks'
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Install & Test') {
            steps {
                sh 'npm ci'
                sh 'npm test'
                sh 'cd server && npm ci && npm test'
            }
        }
        stage('Build Frontend') {
            steps { sh 'npm run build' }
        }
        stage('Generate Image Tag') {
            steps {
                script {
                    def branch = (env.BRANCH_NAME ?: env.GIT_BRANCH?.replaceAll('origin/', '') ?: 'main').replaceAll('/', '-')
                    def timestamp = new Date().format('yyyyMMdd-HHmmss')
                    env.IMAGE_TAG = "${branch}-${timestamp}"
                    echo "Image Tag: ${env.IMAGE_TAG}"
                }
            }
        }
        stage('Docker Build & Push') {
            when {
                anyOf {
                    branch 'develop'
                    branch pattern: 'release*', comparator: 'GLOB'
                    branch 'main'
                    branch pattern: 'hotfix*', comparator: 'GLOB'
                    // Allow regular pipeline jobs (BRANCH_NAME is null)
                    expression { return env.BRANCH_NAME == null }
                }
            }
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                        def ecrRegistry = sh(script: "aws ecr describe-repositories --repository-names ${DOCKER_REPO} --region ${AWS_REGION} --query 'repositories[0].repositoryUri' --output text | sed 's|/${DOCKER_REPO}||'", returnStdout: true).trim()
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ecrRegistry}"
                        sh "docker build -t ${ecrRegistry}/${DOCKER_REPO}:${IMAGE_TAG} ."
                        sh "docker push ${ecrRegistry}/${DOCKER_REPO}:${IMAGE_TAG}"
                        env.ECR_REGISTRY = ecrRegistry
                    }
                }
            }
        }
        stage('Deploy to Dev') {
            when { branch 'develop' }
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                        sed -i 's/name: landmark/name: develop/g' k8s/namespace.yml
                        sed -i 's/namespace: landmark/namespace: develop/g' k8s/*.yml
                        sed -i "s|image: landmark-technologies:latest|image: ${ECR_REGISTRY}/${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                        kubectl apply -f k8s/namespace.yml
                        kubectl apply -f k8s/
                    """ {
            when { branch pattern: 'release*', comparator: 'GLOB' }
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                        sed -i 's/name: landmark/name: staging/g' k8s/namespace.yml
                        sed -i 's/namespace: landmark/namespace: staging/g' k8s/*.yml
                        sed -i "s|image: landmark-technologies:latest|image: ${ECR_REGISTRY}/${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                        kubectl apply -f k8s/namespace.yml
                        kubectl apply -f k8s/
                    """ {
            when {
                anyOf {
                    branch 'main'
                    branch pattern: 'hotfix*', comparator: 'GLOB'
                    expression { return env.BRANCH_NAME == null }
                }
            }
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                        sed -i 's/name: landmark/name: production/g' k8s/namespace.yml
                        sed -i 's/namespace: landmark/namespace: production/g' k8s/*.yml
                        sed -i "s|image: landmark-technologies:latest|image: ${ECR_REGISTRY}/${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                        kubectl apply -f k8s/namespace.yml
                        kubectl apply -f k8s/
                    """
                }
            }
        }
    }
    post {
        success { echo 'Pipeline succeeded!' }
        failure { echo 'Pipeline failed!' }
        always  { cleanWs() }
    }
}
