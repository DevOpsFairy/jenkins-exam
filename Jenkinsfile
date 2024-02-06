pipeline {
    agent any
    
    environment {
        // Credentials
        DOCKER_CREDENTIALS = credentials('DOCKER_HUB_PASS') 
        // Image repositories
        CAST_IMAGE = "bullfinch38/cast-service"
        MOVIE_IMAGE = "bullfinch38/movie-service"
        // Docker Hub Username
        DOCKER_USERNAME = 'bullfinch38' 
    }
    
    stages {
        stage('Checkout code') {
            steps {
                checkout scm
            }
        }

        stage('Build and push images') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'qa'
                    branch 'staging'
                    branch 'main'
                }
            }
            steps {
                script {
                    def dockerTag = env.BRANCH_NAME == 'main' ? 'latest' : env.BRANCH_NAME

                    // Docker login
                    sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_CREDENTIALS}"
                    
                    sh "docker build -t ${env.MOVIE_IMAGE}:${dockerTag} -f ./movie-service/Dockerfile ./movie-service"
                    sh "docker push ${env.MOVIE_IMAGE}:${dockerTag}"

                    sh "docker build -t ${env.CAST_IMAGE}:${dockerTag} -f ./cast-service/Dockerfile ./cast-service"
                    sh "docker push ${env.CAST_IMAGE}:${dockerTag}"

                    // Docker logout
                    sh "docker logout"
                }
            }
        }

        stage('Deploy to Environments') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'qa'
                    branch 'staging'
                }
            }
            environment {
                KUBECONFIG = credentials("config") 
            }
            steps {
                script {                    
                    def valuesFile = "values-${env.BRANCH_NAME}.yaml"
                    // Helm upgrade/install
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config                    
                    '''
                    sh "helm upgrade --install jenkins-devops ./helm-for-jenkins --namespace ${env.BRANCH_NAME} --values ./helm-for-jenkins/${valuesFile} --set image.tag=${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Deployment to Production') {
            when {
                allOf {
                    branch 'main'
                    expression { currentBuild.resultIsBetterOrEqualTo('SUCCESS') }
                }
            }
            environment {
                KUBECONFIG = credentials("config") // Adjust the credential ID accordingly
            }
            steps {
                timeout(time: 5, unit: "MINUTES") {
                    input message: 'Do you want to deploy to production?', ok: 'Yes'
                }
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    '''
                    sh "helm upgrade --install jenkins-devops ./helm-for-jenkins --namespace prod --values ./helm-for-jenkins/values-prod.yaml --set image.tag=${env.BUILD_NUMBER}"
                }
            }
        }
    }

    post {
        always {
            echo 'The pipeline has finished.'
            cleanWs() // Cleans the workspace
        }
        failure {
            echo 'The pipeline failed!'
        }
    }
}
