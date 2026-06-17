pipeline {
  agent any

  environment {
    DOCKERHUB_USER = 'brahimhamdi'  // à remplacer par votre login DockerHub
    IMAGE_TAG      = "${env.BUILD_NUMBER}"
    DOCKERHUB      = credentials('dockerhub')
    STAGING_NS     = 'dockercoins-staging'
    PROD_NS        = 'dockercoins'
  }

  options { timestamps() }

  stages {

    stage('Checkout') {
      steps {
        git url: 'https://github.com/brahimhamdi/dockercoins-cicd.git',
            branch: 'master'
      }
    }

    stage('Build webui') {
      steps {
        sh "docker build -t $DOCKERHUB_USER/webui:$IMAGE_TAG ./webui"
      }
    }

    stage('Smoke test webui') {
      steps {
        sh '''
          docker run -d --rm -p 8003:80 --name webui_smoke $DOCKERHUB_USER/webui:$IMAGE_TAG
          sleep 3
          curl -sf http://localhost:8003/ > /dev/null && echo "webui smoke test OK"
          docker stop webui_smoke
        '''
      }
    }

    stage('Integration test (compose)') {
      steps {
        sh '''
          export WEBUI_IMAGE=$DOCKERHUB_USER/webui:$IMAGE_TAG
          docker compose up -d
          sleep 10
          curl -sf http://localhost:8000/     > /dev/null && echo "integration: webui index OK"
          curl -sf http://localhost:8000/json > /dev/null && echo "integration: /json OK - pile complete cablee"
          docker compose down -v
        '''
      }
    }

    stage('Push webui') {
      steps {
        sh 'echo "$DOCKERHUB_PSW" | docker login -u "$DOCKERHUB_USR" --password-stdin'
        sh "docker push $DOCKERHUB_USER/webui:$IMAGE_TAG"
        sh "docker tag  $DOCKERHUB_USER/webui:$IMAGE_TAG $DOCKERHUB_USER/webui:latest"
        sh "docker push $DOCKERHUB_USER/webui:latest"
      }
    }

    stage('Deploy to staging') {
      steps {
        withKubeConfig([credentialsId: 'kubeconfig']) {
          sh '''
            kubectl create namespace "$STAGING_NS" --dry-run=client -o yaml | kubectl apply -f -
            kubectl apply -f dockercoins.yaml -n "$STAGING_NS"
            kubectl set image deployment/webui webui=$DOCKERHUB_USER/webui:$IMAGE_TAG -n "$STAGING_NS"
            kubectl rollout status deployment/webui -n "$STAGING_NS" --timeout=120s
          '''
        }
      }
    }

    stage('Approve production') {
      steps { input message: 'Deploy to production?', ok: 'Deploy' }
    }

    stage('Deploy to production') {
      steps {
        withKubeConfig([credentialsId: 'kubeconfig']) {
          sh '''
            kubectl delete ns dockercoins-staging --force --grace-period 0
            kubectl create namespace "$PROD_NS" --dry-run=client -o yaml | kubectl apply -f -
            kubectl apply -f dockercoins.yaml -n "$PROD_NS"
            kubectl set image deployment/webui webui=$DOCKERHUB_USER/webui:$IMAGE_TAG -n "$PROD_NS"
            kubectl rollout status deployment/webui -n "$PROD_NS" --timeout=120s
          '''
        }
      }
    }
  }

  post {
    always  {
      sh 'docker compose down -v || true'
      sh 'docker logout || true'
    }
    success { echo "webui $IMAGE_TAG delivered successfully." }
    failure { echo 'Pipeline failed - check the stage view.' }
  }
}
