pipeline {
  agent any

  environment {
    IMAGE_REPO = 'narayanababut/kanban-dashboard'
    APP_NAME   = 'kanban-dashboard'
    EDGE_DIR   = '/opt/kanban-edge'
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

    stage('Blue-Green deploy') {
      steps {
        sh '''
          set -e
          ACTIVE=$(cat ${EDGE_DIR}/active_color 2>/dev/null || echo blue)
          if [ "$ACTIVE" = "blue" ]; then TARGET=green; else TARGET=blue; fi
          echo "Active=$ACTIVE  ->  Target=$TARGET"

          docker pull $IMAGE
          docker rm -f ${APP_NAME}-${TARGET} 2>/dev/null || true
          docker run -d --name ${APP_NAME}-${TARGET} --network web \
            --restart unless-stopped --memory 256m --memory-swap 256m --cpus 0.5 \
            $IMAGE

          # wait until the NEW container is healthy (old one still serving)
          ok=0
          for i in $(seq 1 20); do
            hs=$(docker inspect --format='{{.State.Health.Status}}' ${APP_NAME}-${TARGET} 2>/dev/null || echo starting)
            if [ "$hs" = healthy ] && docker exec edge wget -q --spider http://${APP_NAME}-${TARGET}:8080/healthz; then
              ok=1; echo "target healthy"; break
            fi
            echo "waiting ($hs)"; sleep 3
          done
          [ "$ok" = 1 ] || { echo "target never healthy"; exit 1; }

          # switch traffic: rewrite upstream + graceful reload
          cat > ${EDGE_DIR}/conf.d/upstream.conf <<UPS
upstream kanban_upstream {
    server ${APP_NAME}-${TARGET}:8080;
}
UPS
          docker exec edge nginx -t
          docker exec edge nginx -s reload
          sleep 2
          curl -fsS http://localhost/healthz

          # retire the old color
          echo "$TARGET" > ${EDGE_DIR}/active_color
          docker rm -f ${APP_NAME}-${ACTIVE} 2>/dev/null || true
          echo "Now serving $TARGET"
        '''
      }
    }
  }

  post {
    failure {
      echo 'Deploy failed - proxy untouched, old version still serving'
      sh '''
        ACTIVE=$(cat ${EDGE_DIR}/active_color 2>/dev/null || echo blue)
        if [ "$ACTIVE" = "blue" ]; then TARGET=green; else TARGET=blue; fi
        docker rm -f ${APP_NAME}-${TARGET} 2>/dev/null || true
        echo "Cleaned up failed $TARGET container; $ACTIVE unaffected"
      '''
    }
    success {
      echo "Deployed $IMAGE with zero downtime"
      sh 'docker image prune -f || true'
    }
  }
}
