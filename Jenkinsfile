pipeline {
  environment { 
    DOCKER_ID = "bullfinch38" // replace this with your docker-id
    MOVIES_EXAM_DB = "jenkins-exam-movies-db"
    MOVIES_EXAM_APP = "jenkins-exam-movies-app"
    CASTS_EXAM_APP = "jenkins-exam-casts-app"
    NGINX_EXAM_APP = "jenkins-exam-nginx-app"
    DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build to increment the value by 1 with each new build
  }
  agent any // Jenkins will be able to select all available agents
  stages {
    stage('Cleaning former runs') {
      steps{
        sh """
        docker rm -f movie-db-container
        docker rm -f cast-db-container
        docker rm -f exam-nginx
        docker rm -f exam-movie-app
        docker rm -f exam-casts-app
        docker rm -f exam-test-nginx
        docker rm -f exam-test-movies
        docker rm -f exam-test-casts
        """
      }
    }

    stage('echo branch name'){
      when{
        expression{
          return env.GIT_BRANCH == "origin/master"
        }
      }
      steps{
        echo "master"
      }
    }

    stage('Deploy the movie DB') {
      steps {
        sh '''docker run -d --name movie-db-container -v postgres_data_movie:/var/lib/postgresql/data/ -e POSTGRES_USER=movie_db_username \\
                -e POSTGRES_PASSWORD=movie_db_password -e POSTGRES_DB=movie_db_dev --ip 172.17.0.2 postgres:12.1-alpine'''
      }
    }

    stage('Push images in docker Hub') {
      environment{
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
      }
      steps{
        sh '''
        docker login -u $DOCKER_ID -p $DOCKER_PASS
        docker push $DOCKER_ID/$NGINX_EXAM_APP:$DOCKER_TAG
        docker push $DOCKER_ID/$CASTS_EXAM_APP:$DOCKER_TAG
        docker push $DOCKER_ID/$MOVIES_EXAM_APP:$DOCKER_TAG
        '''
      }
    }

    stage('Deploy Dev') {
      environment{
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from the secret file called config saved on Jenkins
      }
      steps {
        script {
          sh '''
          helm upgrade --install jenkins-dev jenkins-exam/ --values=./jenkins-exam/values.yaml --namespace dev
          '''
        }
      }
    }

    stage('Deploiement en prod') {
      environment{
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from the secret file called config saved on Jenkins
      }
      when{
        expression{
          return env.GIT_BRANCH == "origin/master"
        }
      }
      steps {
        // Create an Approval Button with a timeout of 15 minutes.
        // This requires a manual validation to deploy on the production environment
        timeout(time: 15, unit: "MINUTES"){        
          input message: 'Want to deploy in prod ', ok: 'Yes'
        }                   
      
        script{
          sh '''
          helm upgrade --install jenkins-prod jenkins-exam/ --values=./jenkins-exam/values.yaml --namespace prod
          '''
        }
      }
    }
  }

  post {
    failure {
      echo "This will run if the job failed"
      mail to: "konstantinsnigirev92@gmail.com", // replace this with your email
           subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
           body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
    }
  }
}
