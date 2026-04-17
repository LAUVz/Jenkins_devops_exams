pipeline {
  agent any
  stages {
    stage('Docker Build') {
      steps {
        script {
          sh '''
docker rm -f jenkins || true
docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG .
sleep 6
'''
        }

      }
    }

    stage('Docker Run') {
      steps {
        script {
          sh '''
docker run -d -p 80:80 --name jenkins $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
sleep 10
'''
        }

      }
    }

    stage('Test Acceptance') {
      steps {
        script {
          sh 'curl localhost'
        }

      }
    }

    stage('Docker Push') {
      environment {
        DOCKER_PASS = credentials('DOCKER_HUB_PASS')
      }
      steps {
        script {
          sh '''
docker login -u $DOCKER_ID -p $DOCKER_PASS
docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
'''
        }

      }
    }

    stage('Deploy to dev') {
      environment {
        KUBECONFIG = credentials('config')
      }
      steps {
        script {
          sh '''
rm -Rf .kube
mkdir .kube
cat $KUBECONFIG > .kube/config
cp fastapi/values.yaml values.yml
sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
helm upgrade --install app fastapi --values=values.yml --namespace dev
'''
        }

      }
    }

    stage('Deploy to QA') {
      environment {
        KUBECONFIG = credentials('config')
      }
      steps {
        script {
          sh '''
rm -Rf .kube
mkdir .kube
cat $KUBECONFIG > .kube/config
cp fastapi/values.yaml values.yml
sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
helm upgrade --install app fastapi --values=values.yml --namespace qa
'''
        }

      }
    }

    stage('Deploy to staging') {
      environment {
        KUBECONFIG = credentials('config')
      }
      steps {
        script {
          sh '''
rm -Rf .kube
mkdir .kube
cat $KUBECONFIG > .kube/config
cp fastapi/values.yaml values.yml
sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
helm upgrade --install app fastapi --values=values.yml --namespace staging
'''
        }

      }
    }

    stage('Deploy to prod') {
      when {
        branch 'master'
      }
      environment {
        KUBECONFIG = credentials('config')
      }
      steps {
        timeout(time: 15, unit: 'MINUTES') {
          input(message: 'Deploy to production?', ok: 'Yes')
        }

        script {
          sh '''
rm -Rf .kube
mkdir .kube
cat $KUBECONFIG > .kube/config
cp fastapi/values.yaml values.yml
sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
helm upgrade --install app fastapi --values=values.yml --namespace prod
'''
        }

      }
    }

  }
  environment {
    DOCKER_ID = 'henrijskons'
    DOCKER_IMAGE = 'datascientestapi'
    DOCKER_TAG = "v.${BUILD_ID}.0"
  }
  post {
    failure {
      echo "Pipeline failed - Build #${env.BUILD_ID}"
      mail(to: 'henrijskons@gmail.com', subject: "${env.JOB_NAME} - Build #${env.BUILD_ID} failed", body: "Console output: ${env.BUILD_URL}")
    }

  }
}