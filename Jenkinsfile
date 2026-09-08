pipeline {
  agent any

  environment {
    IMAGE_REPO  = 'narayanababut/kanban-dashboard'
    APP_NAME    = 'kanban-dashboard'
    APP_PORT    = '80'
    CONT_PORT   = '8080'
    CANARY_PORT = '8081'
  }

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.GIT_SHA   = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
          env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_SHA}"
          env.IMAGE     = "${env.IMAGE_REPO}:${env.IMAGE_TAG}"
        }
        echo "Building ${env.IMAGE}"
      }
    }

    stage('Build image') {
      steps { sh 'docker build -t $IMAGE -t $IMAGE_REPO:latest .' }
    }

    stage('Scan image') {
      steps {
        sh '''
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image --exit-code 0 \
            --severity HIGH,CRITICAL --no-progress $IMAGE || true
        '''
      }
    }

    stage('Push to registry') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub',
              usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
          sh '''
            printf '%s' "$REG_PASS" | docker login -u "$REG_USER" --password-stdin
            docker push $IMAGE
            docker push $IMAGE_REPO:latest
            docker logout
          '''
        }
      }
    }

    stage('Deploy (canary alongside old)') {
      steps {
        sh '''
          docker inspect --format='{{.Config.Image}}' $APP_NAME > .old_image 2>/dev/null || echo none > .old_image
          echo "Previous image: $(cat .old_image)"
          docker pull $IMAGE
          docker rm -f ${APP_NAME}-canary 2>/dev/null || true
          docker run -d --name ${APP_NAME}-canary \
            --restart unless-stopped --memory 256m --memory-swap 256m --cpus 0.5 \
            -p ${CANARY_PORT}:${CONT_PORT} $IMAGE
        '''
      }
    }

    stage('Health check canary') {
      steps {
        sh '''
          ok=0
          for i in $(seq 1 15); do
            if curl -fsS http://localhost:${CANARY_PORT}/healthz >/dev/null; then ok=1; break; fi
            sleep 2
          done
          [ "$ok" = 1 ] || { echo "canary failed HTTP check"; exit 1; }
          for i in $(seq 1 10); do
            hs=$(docker inspect --format='{{.State.Health.Status}}' ${APP_NAME}-canary)
            echo "HEALTHCHECK: $hs"; [ "$hs" = healthy ] && break; sleep 3
          done
          [ "$hs" = healthy ] || { echo "canary not healthy"; exit 1; }
        '''
      }
    }

    stage('Promote to production') {
      steps {
        sh '''
          docker rm -f $APP_NAME 2>/dev/null || true
          docker run -d --name $APP_NAME \
            --restart unless-stopped --memory 256m --memory-swap 256m --cpus 0.5 \
            -p ${APP_PORT}:${CONT_PORT} $IMAGE
          sleep 3
          curl -fsS http://localhost:${APP_PORT}/healthz
          docker rm -f ${APP_NAME}-canary 2>/dev/null || true
        '''
      }
    }
  }

  post {
    failure {
      echo 'Pipeline failed - rolling back'
      sh '''
        OLD=$(cat .old_image 2>/dev/null || echo none)
        docker rm -f ${APP_NAME}-canary 2>/dev/null || true
        if [ "$OLD" != none ] && [ -n "$OLD" ]; then
          docker rm -f $APP_NAME 2>/dev/null || true
          docker run -d --name $APP_NAME \
            --restart unless-stopped --memory 256m --memory-swap 256m --cpus 0.5 \
            -p ${APP_PORT}:${CONT_PORT} $OLD
          echo "Rolled back to $OLD"
        fi
      '''
    }
    success {
      echo "Deployed $IMAGE"
      sh 'docker image prune -f || true'
    }
  }
}
