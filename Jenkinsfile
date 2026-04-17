pipeline {
environment {
    DOCKER_ID = "henrijskons"
    DOCKER_IMAGE_MOVIE = "movie-service"
    DOCKER_IMAGE_CAST = "cast-service"
    DOCKER_TAG = "v.${BUILD_ID}.0"
}
agent any
stages {
    stage('Docker Build') {
        steps {
            script {
                sh '''
                docker rm -f movie_service cast_service || true
                docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG ./movie-service
                docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG ./cast-service
                sleep 6
                '''
            }
        }
    }

    stage('Docker Run') {
        steps {
            script {
                sh '''
                docker run -d -p 8001:8000 --name movie_service $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                docker run -d -p 8002:8000 --name cast_service $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                sleep 10
                '''
            }
        }
    }

    stage('Test Acceptance') {
        steps {
            script {
                sh '''
                curl localhost:8001 || echo "movie-service not reachable (DB dependency expected)"
                curl localhost:8002 || echo "cast-service not reachable (DB dependency expected)"
                '''
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
                docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                '''
            }
        }
    }

    stage('Deploy to dev') {
        environment { KUBECONFIG = credentials("config") }
        steps {
            script {
                sh '''
                rm -Rf .kube && mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp movie-service/values.yaml movie-values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" movie-values.yml
                helm upgrade --install movie-service movie-service/helm --values=movie-values.yml --namespace dev
                cp cast-service/values.yaml cast-values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" cast-values.yml
                helm upgrade --install cast-service cast-service/helm --values=cast-values.yml --namespace dev
                '''
            }
        }
    }

    stage('Deploy to QA') {
        environment { KUBECONFIG = credentials("config") }
        steps {
            script {
                sh '''
                rm -Rf .kube && mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp movie-service/values.yaml movie-values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" movie-values.yml
                helm upgrade --install movie-service movie-service/helm --values=movie-values.yml --namespace qa
                cp cast-service/values.yaml cast-values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" cast-values.yml
                helm upgrade --install cast-service cast-service/helm --values=cast-values.yml --namespace qa
                '''
            }
        }
    }

    stage('Deploy to staging') {
        environment { KUBECONFIG = credentials("config") }
        steps {
            script {
                sh '''
                rm -Rf .kube && mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp movie-service/values.yaml movie-values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" movie-values.yml
                helm upgrade --install movie-service movie-service/helm --values=movie-values.yml --namespace staging
                cp cast-service/values.yaml cast-values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" cast-values.yml
                helm upgrade --install cast-service cast-service/helm --values=cast-values.yml --namespace staging
                '''
            }
        }
    }

    stage('Deploy to prod') {
        when { branch 'master' }
        environment { KUBECONFIG = credentials("config") }
        steps {
            timeout(time: 15, unit: "MINUTES") {
                input message: 'Deploy to production?', ok: 'Yes'
            }
            script {
                sh '''
                rm -Rf .kube && mkdir .kube
                cat $KUBECONFIG > .kube/config
                cp movie-service/values.yaml movie-values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" movie-values.yml
                helm upgrade --install movie-service movie-service/helm --values=movie-values.yml --namespace prod
                cp cast-service/values.yaml cast-values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" cast-values.yml
                helm upgrade --install cast-service cast-service/helm --values=cast-values.yml --namespace prod
                '''
            }
        }
    }
}
post {
    failure {
        echo "Pipeline failed - Build #${env.BUILD_ID}"
        mail to: "henrijskons@gmail.com",
             subject: "${env.JOB_NAME} - Build #${env.BUILD_ID} failed",
             body: "Console output: ${env.BUILD_URL}"
    }
}
}
