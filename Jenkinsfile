pipeline {
   agent any
   environment {
     DOCKER_REGISTRY = "registry.devops:5000"
     SECURE_LOG_LEVEL = "debug"
     LOCAL_MACHINE_IP_ADDRESS="jenkins.devops"
   }
   stages {
      stage('Build') {
         steps {
            sh '''
               docker images -f dangling=true -q | xargs docker rmi || true
            '''   
         }
      }
      stage('Code Security') {
         steps {
            parallel(
               dependency: {
                  sh '''
                     docker run --env SECURE_LOG_LEVEL=${SECURE_LOG_LEVEL} -v "$PWD:/code" -v /var/run/docker.sock:/var/run/docker.sock registry.gitlab.com/gitlab-org/security-products/dependency-scanning:latest /code
                  '''
               },
               sast: {
                  // echo 'SAST'
                  sh '''
                     docker run --volume "$PWD":/code --volume /var/run/docker.sock:/var/run/docker.sock registry.gitlab.com/gitlab-org/security-products/sast:latest /app/bin/run /code
                  '''
               }
            )
         }
      }
      stage('Staging Setup') {
         steps {
            sh '''
               docker build --no-cache --build-arg STAGE=staging -t "devops/webapp:staging" .
               docker tag "devops/webapp:staging" "${DOCKER_REGISTRY}/devops/webapp:staging"
               docker push "${DOCKER_REGISTRY}/devops/webapp:staging"
               docker rmi "${DOCKER_REGISTRY}/devops/webapp:staging"
               docker rmi "devops/webapp:staging"
            '''
            script {
                     def remote = [:]
                     remote.name = 'staging'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'staging.devops'
                     remote.identityFile = '~/.ssh/staging.key'
                     sshCommand remote: remote, command: "docker stop webapp || true"
                     sshCommand remote: remote, command: "docker rm webapp || true"
                     sshCommand remote: remote, command: "docker rmi ${DOCKER_REGISTRY}/devops/webapp:staging || true"
                  }
         }
      }
      stage('Staging Deploy') {//providing delay for mysql to start
         steps {   
            script {
                def remote = [:]
                remote.name = 'staging'
                remote.user = 'vagrant'
                remote.allowAnyHosts = true
                remote.host = 'staging.devops'
                remote.identityFile = '~/.ssh/staging.key'
                sshCommand remote: remote, command: "docker run -d -p 80:5000 --name webapp ${DOCKER_REGISTRY}/devops/webapp:staging"                  
            }
         }
      }
      stage('UAT Tests') {
         steps {   
               echo 'UAT Tests'
         }
      }
      stage('DAST') {
         steps {   
               sh '''
                  mkdir wrk
                  chmod 777 wrk
                  docker run \
                     --volume $(pwd)/wrk:/output:rw \
                     --volume $(pwd)/wrk:/zap/wrk:rw \
                     registry.gitlab.com/gitlab-org/security-products/dast:latest /analyze -t http://staging.devops -r report.html
               '''
         }
      }
      stage('Production Setup') {
         steps {
            sh '''
               docker build --no-cache --build-arg STAGE=prod -t "devops/webapp:prod" .
               docker tag "devops/webapp:prod" "${DOCKER_REGISTRY}/devops/webapp:prod"
               docker push "${DOCKER_REGISTRY}/devops/webapp:prod"
               docker rmi "${DOCKER_REGISTRY}/devops/webapp:prod"
               docker rmi "devops/webapp:prod"
            '''
            script {
                     def remote = [:]
                     remote.name = 'production'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'production.devops'
                     remote.identityFile = '~/.ssh/production.key'
                     sshCommand remote: remote, command: "docker stop webapp || true"
                     sshCommand remote: remote, command: "docker rm webapp || true"
                     sshCommand remote: remote, command: "docker rmi ${DOCKER_REGISTRY}/devops/webapp:prod || true"
                  }
         }
      }
      stage('Infrastructure Scan') {
         steps {   
               sh '''
                  docker stop clair-db || true
                  docker rm clair-db || true
                  docker run -p 5432:5432 -d --name clair-db arminc/clair-db:latest
                  docker run \
                  --interactive --rm \
                  --volume "$PWD":/tmp/app \
                  -e CI_PROJECT_DIR=/tmp/app \
                  -e CLAIR_DB_CONNECTION_STRING="postgresql://postgres:password@${LOCAL_MACHINE_IP_ADDRESS}:5432/postgres?sslmode=disable&statement_timeout=60000" \
                  -e CI_APPLICATION_REPOSITORY=${DOCKER_REGISTRY}/devops/webapp \
                  -e CI_APPLICATION_TAG=prod \
                  -e REGISTRY_INSECURE=true \
                  registry.gitlab.com/gitlab-org/security-products/analyzers/klar
               '''
         }
      }
      stage('Production Deploy Approval') {
         steps {
            script {
                  input message: 'Do you approve Deployment ?', ok: 'OK'
            }
         }
      }      
      stage('Production Deploy') {       
         steps {   
            script {
                     def remote = [:]
                     remote.name = 'production'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'production.devops'
                     remote.identityFile = '~/.ssh/production.key'
                     sshCommand remote: remote, command: "docker run -d -p 80:5000 --name webapp ${DOCKER_REGISTRY}/devops/webapp:prod"
                  }
         }
      }                       
   }
   post {
    failure {
      script {
        currentBuild.result = 'FAILURE'
      }
    }
   //  always {
   //    step([$class: 'WsCleanup'])
   //  }
  }
}