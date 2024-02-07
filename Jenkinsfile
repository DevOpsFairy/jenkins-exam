pipeline {
environment { // Declaration of environment variables
DOCKER_ID = "bullfinch38" // replace this with your docker-id
DOCKER_IMAGE_CAST = "cast-image"
DOCKER_IMAGE_MOVIE = "movie-image"
DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
}
agent any // Jenkins will be able to select all available agents
stages {

        stage(' Docker run databases '){
            parallel {   
                stage(' Docker run movie db'){ // run container from our builded image
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
                stage(' Docker run cast db'){ // run container from our builded image
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

        stage(' Docker Build'){
            parallel {
                stage(' Docker Build Cast'){ // docker build image stage
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
                stage(' Docker Build Movie'){ // docker build image stage
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
         stage(' Docker run'){
            parallel {   
                stage(' Docker run Cast'){ // run container from our builded image
                        steps {
                            script {
                            sh '''
                            docker run -d -p 8002:8000 --volume ./cast-service/:/app/ -e DATABASE_URI='postgresql://cast_db_username:cast_db_password@cast_db/cast_db_dev' --name cast_service $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG 
                            sleep 10
                            '''
                            }
                        }
                }
                stage(' Docker run Movie'){ // run container from our builded image
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

         stage(' Docker run ngnix '){
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

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                    curl localhost
                    '''
                    }
            }

        }
        stage('Docker Push'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
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
        
        stage('Deployment db ti dev'){
            parallel {
                stage('Deployment to dev cast db app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp castdb/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install castdb castdb --values=values.yml --namespace dev
                                '''
                                }
                            }

                }

                stage('Deployment to dev movie db app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp moviedb/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install moviedb moviedb --values=values.yml --namespace dev
                                '''
                                }
                            }

                }
            }
        }        

        stage('Deployment app to dev'){

            parallel {
                stage('Deployment to dev cast app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp cast/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install cast cast --values=values.yml --namespace dev
                                '''
                                }
                            }

                }

                stage('Deployment to dev movie app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp movie/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install movie movie --values=values.yml --namespace dev
                                '''
                                }
                            }

                }
            }


        }

        stage('Deployment to dev nginx app'){
            environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp nginx/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install nginx nginx --values=values.yml --namespace dev
                                '''
                                }
                            }
        }            

        stage(' Deployment db to qa'){
            parallel {
                stage('Deployment to qa cast db app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp castdb/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install castdb castdb --values=values.yml --namespace qa
                                '''
                                }
                            }

                }

                stage('Deployment to qa movie db app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp moviedb/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install moviedb moviedb --values=values.yml --namespace qa
                                '''
                                }
                            }

                }
            }
        }        

        stage(' Deployment app to qa'){
            parallel {
                stage('Deployment to qa cast app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp cast/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install cast cast --values=values.yml --namespace qa
                                '''
                                }
                            }

                }

                stage('Deployment to qa movie app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp movie/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install movie movie --values=values.yml --namespace qa
                                '''
                                }
                            }

                }
            }
        }

          stage('Deployment to qa nginx app'){
            environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp nginx/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install nginx nginx --values=values.yml --namespace qa
                                '''
                                }
                            }
          }           


        stage(' Deployment db to staging'){
            parallel {
                stage('Deployment to staging cast db app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp castdb/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install castdb castdb --values=values.yml --namespace staging
                                '''
                                }
                            }

                }

                stage('Deployment to staging movie db app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp moviedb/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install moviedb moviedb --values=values.yml --namespace staging
                                '''
                                }
                            }

                }
            }
        }        

        stage(' Deployment to staging'){
            parallel {
                stage('Deployment to staging cast app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp cast/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install cast cast --values=values.yml --namespace staging
                                '''
                                }
                            }

                }

                stage('Deployment to staging movie app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp movie/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install movie movie --values=values.yml --namespace staging
                                '''
                                }
                            }

                }
            }
        }

          stage('Deployment to staging nginx app'){
            environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp nginx/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install nginx nginx --values=values.yml --namespace staging
                                '''
                                }
                            }
        }    

        stage(' Deployment db to prod'){
            parallel {
                stage('Deployment to prod cast db app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                timeout(time: 15, unit: "MINUTES") {
                                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                                }
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp castdb/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install castdb castdb --values=values.yml --namespace prod
                                '''
                                }
                            }

                }

                stage('Deployment to prod movie db app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                timeout(time: 15, unit: "MINUTES") {
                                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                                }
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp moviedb/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install moviedb moviedb --values=values.yml --namespace prod
                                '''
                                }
                            }

                }
            }
        }

        stage(' Deployment to prod'){
            parallel {
                stage('Deployment to prod cast app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                timeout(time: 15, unit: "MINUTES") {
                                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                                }
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp cast/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install cast cast --values=values.yml --namespace prod
                                '''
                                }
                            }

                }

                stage('Deployment to prod movie app'){
                        environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                timeout(time: 15, unit: "MINUTES") {
                                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                                }
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp movie/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install movie movie --values=values.yml --namespace prod
                                '''
                                }
                            }

                }
            }
        }

        stage('Deployment to prod nginx app'){
            environment
                        {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                        }
                            steps {
                                script {
                                sh '''
                                rm -Rf .kube
                                mkdir .kube
                                ls
                                cat $KUBECONFIG > .kube/config
                                cp nginx/values.yaml values.yml
                                cat values.yml
                                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                                helm upgrade --install nginx nginx --values=values.yml --namespace prod
                                '''
                                }
                            }
        }    
}
}