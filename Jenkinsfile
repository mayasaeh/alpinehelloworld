/* import shared library */
@Library('ynov-slackNotifier')_
pipeline{
  environment{
    IMAGE_NAME = "mayas213/alpinehelloworld"
    IMAGE_TAG = "${BUILD_TAG}"
    CONTAINER_NAME = "alpinehelloworld"
    STAGING = "ynov-mayas-staging"
    PRODUCTION = "ynov-mayas-production"
    DOCKERHUB_PASSWORD = credentials('dockerhub_password')
    PRODUCTION_HOST = "52.207.214.244"
  }
  agent none
  
  stages{
    
    stage ('Build Stage'){
      agent any
      steps{
        script{
          sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
        }
      }
    }
      
    stage ('Run Container based on builded image'){
      agent any
      steps{
        script{
          sh '''
            docker run -d --name ${CONTAINER_NAME} -e PORT=5000 -p 5000:5000 ${IMAGE_NAME}:${IMAGE_TAG}
          '''
        }
      }
    }
    
    stage ('Test image'){
      agent any
      steps{
        script{
          sh '''
            curl localhost:5000 | grep -q "Hello world!"
          '''
        }
      }
    }
    
    stage ('Delete Container'){
      agent any
      steps{
        script{
          sh '''
            docker stop ${CONTAINER_NAME}
            docker rm ${CONTAINER_NAME}
          '''
        }
      }
    }
    
    stage ('Push image and deploy it in staging env'){
      when{
        expression {GIT_BRANCH == 'origin/master'}
      }
      agent any
      environment{
        HEROKU_API_KEY = credentials('heroku_api_key')
      }
      steps{
        script{
          sh'''
            heroku container:login
            heroku create ${STAGING} || echo "This env already exist"
            heroku container:push -a ${STAGING} web
            heroku container:release -a ${STAGING} web
          '''
        }
      }
    }

    stage ('Push image on DockerHub'){
        agent any
        steps{
            script{
            sh '''
                docker login -u mayas213 -p ${DOCKERHUB_PASSWORD}
                docker push ${IMAGE_NAME}:${IMAGE_TAG}
                docker rmi ${IMAGE_NAME}:${IMAGE_TAG}
            '''
        }
      }
    }
    
    stage ('Push image and deploy it in production env'){
      when{
        expression {GIT_BRANCH == 'origin/master'}
      }
      agent any
      environment{
        HEROKU_API_KEY = credentials('heroku_api_key')
      }
      steps{
        script{
          sh'''
            heroku container:login
            heroku create ${PRODUCTION} || echo "This env already exist"
            heroku container:push -a ${PRODUCTION} web
            heroku container:release -a ${PRODUCTION} web
          '''
        }
      }
    }

    stage ('Run container on PROD HOST'){
      agent { label 'prod'}
      steps{
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          script{
            sh '''
              sudo docker stop ${CONTAINER_NAME}
              sudo docker rm ${CONTAINER_NAME}
              sudo docker run -d --name ${CONTAINER_NAME} -e PORT=5000 -p 80:5000 ${IMAGE_NAME}:${IMAGE_TAG}
              sleep 5
              curl http://localhost:80 | grep -q "Hello world!"
              curl http://${PRODUCTION_HOST}:80 | grep -q "Hello world!"
            '''
          }
        }
      }
    }   
  }
  post {
    always{
      script{
        slackNotifier currentBuild.result
      }
    }
  }
}
