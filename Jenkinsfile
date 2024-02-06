pipeline {
    environment {
        DOCKER_ID = "bullfinch38"
        DOCKER_IMAGE = "movie-image"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        KUBE_NAMESPACE_DEV = "dev"
        KUBE_NAMESPACE_QA = "qa"
        KUBE_NAMESPACE_STAGING = "staging"
        KUBE_NAMESPACE_PROD = "prod"
        HELM_CHART_NAME = "helm-for-jenkins"
    }

    agent any

    stages {
        stage('Build and Push Docker Images') {
            steps {
                script {
                    docker.build("${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}", "./")
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_credentials') {
                        docker.image("${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                    }
                }
            }
        }

        stage('Unit Test - Movies API') {
            steps {
                script {
                    sh '''
                        docker run --rm --name test-movies-api -v ./movie-service/:/app/ $DOCKER_ID/${DOCKER_IMAGE}:${DOCKER_TAG} pytest -v
                    '''
                }
            }
        }

        stage('Unit Test - Casts API') {
            steps {
                script {
                    sh '''
                        docker run --rm --name test-casts-api -v ./cast-service/:/app/ $DOCKER_ID/${DOCKER_IMAGE}:${DOCKER_TAG} pytest -v
                    '''
                }
            }
        }

        stage('Integration Test - Nginx') {
            steps {
                script {
                    sh '''
                        docker run --rm --name test-nginx -v ./nginx_config.conf:/etc/nginx/default.conf curlimages/curl -L -v http://172.17.0.4
                    '''
                }
            }
        }

        stage('Integration Test - Movies API') {
            steps {
                script {
                    sh '''
                        docker run --rm --name test-movies-api curlimages/curl -L -v http://172.17.0.5:8000/api/v1/movies/status
                    '''
                }
            }
        }

        stage('Integration Test - Casts API') {
            steps {
                script {
                    sh '''
                        docker run --rm --name test-casts-api curlimages/curl -L -v http://172.17.0.6:8000/api/v1/casts/status
                    '''
                }
            }
        }

        stage('Push images in Docker Hub') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                sh '''
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    docker push $DOCKER_ID/${DOCKER_IMAGE}:${DOCKER_TAG}
                '''
            }
        }

        stage('Deploy to Dev Environment') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        sh '''
                            cp ${KUBECONFIG} .kube/config
                            helm upgrade --install app ${HELM_CHART_NAME} --namespace ${KUBE_NAMESPACE_DEV} --set movie.image=${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG} --set cast.image=${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}
                        '''
                    }
                }
            }
        }

        stage('Deploy to QA Environment') {
            // Add QA deployment steps here
        }

        stage('Deploy to Staging Environment') {
            // Add Staging deployment steps here
        }

        stage('Deploy to Prod Environment') {
            when {
                expression {
                    return env.GIT_BRANCH == "origin/master"
                }
            }
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Want to deploy in prod ', ok: 'Yes'
                }

                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        sh '''
                            cp ${KUBECONFIG} .kube/config
                            helm upgrade --install app ${HELM_CHART_NAME} --namespace ${KUBE_NAMESPACE_PROD} --set movie.image=${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG} --set cast.image=${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}
                        '''
                    }
                }
            }
        }
    }

    post {
        failure {
            mail to: "konstantinsnigirev92@gmail.com",
                 subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
                 body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
        }
    }
}
