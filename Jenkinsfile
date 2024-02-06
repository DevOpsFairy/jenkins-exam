pipeline {
    environment {
        DOCKER_ID = "bullfinch38"
        CAST_IMAGE = "cast-service"
        MOVIE_IMAGE = "movie-service"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }

    agent any

    stages {
        stage('Docker Build') {
            steps {
                script {
                    sh '''
                        docker build -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG ./cast-service
                        sleep 6
                    '''
                    sh '''
                        docker build -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG ./movie-service
                        sleep 6
                    '''
                }
            }
        }

        stage('Docker Run') {
            steps {
                script {
                    sh '''
                        docker run -d --name cast-service $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
                        sleep 10
                    '''
                    sh '''
                        docker run -d --name movie-service -p 8080:8080 $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
                        sleep 10
                    '''
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    sh 'curl localhost:8080'
                    sh 'curl localhost:8000'
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }

            steps {
                script {
                    sh '''
                        docker login -u $DOCKER_ID -p $DOCKER_PASS
                        docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
                        docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Deployment in dev') {
            environment {
                KUBECONFIG = credentials("config")
            }

            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        cp helm/values.yaml values.yml
                        sed -i "s+castImageTag.*+castImageTag: ${DOCKER_TAG}+g" values.yml
                        sed -i "s+movieImageTag.*+movieImageTag: ${DOCKER_TAG}+g" values.yml
                        helm upgrade --install app helm --values=values.yml --namespace dev
                    '''
                }
            }
        }

        stage('Deployment in staging') {
            environment {
                KUBECONFIG = credentials("config")
            }

            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        cp helm/values.yaml values.yml
                        sed -i "s+castImageTag.*+castImageTag: ${DOCKER_TAG}+g" values.yml
                        sed -i "s+movieImageTag.*+movieImageTag: ${DOCKER_TAG}+g" values.yml
                        helm upgrade --install app helm --values=values.yml --namespace staging
                    '''
                }
            }
        }

        stage('Deployment in prod') {
            when {
                expression { env.BRANCH_NAME == 'main' }
            }

            environment {
                KUBECONFIG = credentials("config")
            }

            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production?', ok: 'Yes'
                }

                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        cp helm/values.yaml values.yml
                        sed -i "s+castImageTag.*+castImageTag: ${DOCKER_TAG}+g" values.yml
                        sed -i "s+movieImageTag.*+movieImageTag: ${DOCKER_TAG}+g" values.yml
                        helm upgrade --install app helm --values=values.yml --namespace prod
                    '''
                }
            }
        }
    }

    post {
        success {
            sh '''
                docker stop cast-service movie-service
                docker rm cast-service movie-service
            '''
        }

        failure {
            echo "This will run if the job failed"
            mail to: "konstantinsnigirev92@gmail.com",
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
                body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
        }
    }
}
