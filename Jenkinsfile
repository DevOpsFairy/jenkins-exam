pipeline {
    environment {
        // Declaration of environment variables
        DOCKER_ID = 'bullfinch38' // replace this with your docker-id
        DOCKER_IMAGE_CAST = 'cast-image'
        DOCKER_IMAGE_MOVIE = 'movie-image'
        DOCKER_TAG = "v.${BUILD_ID}.0" // tag images with the current build to increment the value with each new build
    }
    agent any // Jenkins can select all available agents

    stages {
        stage(' Docker run databases ') {
            parallel {
                stage(' Docker run movie db') {
                    steps {
                        script {
                            sh '''
                            docker rm -f movie_db
                            docker run -d --volume postgres_data_movie:/var/lib/postgresql/data/ -e POSTGRES_USER='movie_db_username' -e POSTGRES_PASSWORD='movie_db_password' -e POSTGRES_DB='movie_db_dev' --name movie_db postgres:12.1-alpine
                            sleep 10
                            '''
                        }
                    }
                }
                stage(' Docker run cast db') {
                    steps {
                        script {
                            sh '''
                            docker rm -f cast_db
                            docker run -d --volume postgres_data_cast:/var/lib/postgresql/data/ -e POSTGRES_USER='cast_db_username' -e POSTGRES_PASSWORD='cast_db_password' -e POSTGRES_DB='cast_db_dev' --name cast_db postgres:12.1-alpine
                            sleep 10
                            '''
                        }
                    }
                }
            }
        }

        stage(' Docker Build') {
            parallel {
                stage(' Docker Build Cast') {
                    steps {
                        script {
                            sh '''
                            docker rm -f cast_service
                            docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG ./cast-service/
                            sleep 6
                            '''
                        }
                    }
                }
                stage(' Docker Build Movie') {
                    steps {
                        script {
                            sh '''
                            docker rm -f movie_service
                            docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG ./movie-service/
                            sleep 6
                            '''
                        }
                    }
                }
            }
        }

        stage(' Docker run') {
            parallel {
                stage(' Docker run Cast') {
                    steps {
                        script {
                            sh '''
                            docker run -d -p 8002:8000 --volume ./cast-service/:/app/ -e DATABASE_URI='postgresql://cast_db_username:cast_db_password@cast_db/cast_db_dev' --name cast_service $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                            sleep 10
                            '''
                        }
                    }
                }
                stage(' Docker run Movie') {
                    steps {
                        script {
                            sh '''
                            docker run -d -p 8001:8000 --volume ./movie-service/:/app/ -e DATABASE_URI='postgresql://movie_db_username:movie_db_password@movie_db/movie_db_dev' -e CAST_SERVICE_HOST_URL='http://cast_service:8000/api/v1/casts/' --name movie_service $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                            sleep 10
                            '''
                        }
                    }
                }
            }
        }

        stage(' Docker run ngnix ') {
            steps {
                script {
                    sh '''
                    docker rm -f nginx
                    docker run -d -p 8088:8080 --volume ./nginx_config.conf:/etc/nginx/conf.d/default.conf --name nginx nginx:latest
                    sleep 10
                    '''
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                    curl localhost
                    '''
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials('DOCKER_HUB_PASS') // retrieve docker password from secret text called docker_hub_pass saved on jenkins
            }

            steps {
                script {
                    sh '''
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                    docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Deployment to dev') {
            parallel {
                stage('Deployment to dev') {
                    environment {
                        KUBECONFIG = credentials('config') // retrieve kubeconfig from secret file called config saved on jenkins
                    }
                    steps {
                        script {
                            sh '''
                            rm -Rf .kube
                            mkdir .kube
                            ls
                            cat $KUBECONFIG > .kube/config
                            cp helm-for-jenkins/values-dev.yaml values-dev.yml
                            cat values-dev.yml
                            sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-dev.yml
                            helm upgrade --install exam helm-for-jenkins --values=values-dev.yml --namespace dev
                            '''
                        }
                    }
                }
            }
        }

        stage(' Deployment to qa') {
            parallel {
                stage('Deployment to qa') {
                    environment {
                        KUBECONFIG = credentials('config') // retrieve kubeconfig from secret file called config saved on jenkins
                    }
                    steps {
                        script {
                            sh '''
                            rm -Rf .kube
                            mkdir .kube
                            ls
                            cat $KUBECONFIG > .kube/config
                            cp helm-for-jenkins/values-qa.yaml values-qa.yml
                            cat values-qa.yml
                            sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-qa.yml
                            helm upgrade --install exam helm-for-jenkins --values=values-qa.yml --namespace qa
                            '''
                        }
                    }
                }
            }
        }

        stage(' Deployment to staging') {
            parallel {
                stage('Deployment to staging') {
                    environment {
                        KUBECONFIG = credentials('config') // retrieve kubeconfig from secret file called config saved on jenkins
                    }
                    steps {
                        script {
                            sh '''
                            rm -Rf .kube
                            mkdir .kube
                            ls
                            cat $KUBECONFIG > .kube/config
                            cp helm-for-jenkins/values-staging.yaml values-staging.yml
                            cat values-staging.yml
                            sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-staging.yml
                            helm upgrade --install exam helm-for-jenkins --values=values-staging.yml --namespace staging
                            '''
                        }
                    }
                }
            }
        }

        stage(' Deployment to staging') {
            parallel {
                stage('Deployment to staging') {
                    environment {
                        KUBECONFIG = credentials('config') // retrieve kubeconfig from secret file called config saved on jenkins
                    }
                    steps {
                        script {
                            sh '''
                            rm -Rf .kube
                            mkdir .kube
                            ls
                            cat $KUBECONFIG > .kube/config
                            cp helm-for-jenkins/values-staging.yaml values-staging.yml
                            cat values-staging.yml
                            sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-staging.yml
                            helm upgrade --install exam helm-for-jenkins --values=values-staging.yml --namespace staging
                            '''
                        }
                    }
                }
            }
        }

        stage(' Deployment to prod') {
            parallel {
                stage('Deployment to prod') {
                    environment {
                        KUBECONFIG = credentials('config') // retrieve kubeconfig from secret file called config saved on jenkins
                    }
                    steps {
                        timeout(time: 15, unit: 'MINUTES') {
                            input message: 'Do you want to deploy in production ?', ok: 'Yes'
                        }
                        script {
                            sh '''
                            rm -Rf .kube
                            mkdir .kube
                            ls
                            cat $KUBECONFIG > .kube/config
                            cp helm-for-jenkins/values-prod.yaml values-prod.yml
                            cat values-prod.yml
                            sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values-prod.yml
                            helm upgrade --install exam helm-for-jenkins --values=values-prod.yml --namespace prod
                            '''
                        }
                    }
                }
            }
        }
    }
}
