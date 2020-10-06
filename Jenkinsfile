pipeline {
   agent any
   environment {
     DOCKER_REGISTRY = "registry.devops:5000"
     SECURE_LOG_LEVEL = "debug"
     LOCAL_MACHINE_IP_ADDRESS="jenkins.devops"
     ARCHERYSEC_HOST ="http://archerysec.devops" // ArcherySec URL
     TARGET_URL="http://staging.devops"
     // These secrets should be use through Jenkins Secrets in production implementation
     // ARCHERYSEC_USER = "admin" // archerysec username
     // ARCHERYSEC_PASS = "devops@123A" // archerysec password

     // Archerysec setup
     PROJECT_NAME="devsecops"
     PROJECT_DISC="devsecops project"
     PROJECT_OWNER="Dev"
   }
   stages {
      stage('Build') {
         steps {
            sh '''
            # docker image and container clean up
               docker system prune -a -f
               docker images -f dangling=true -q | xargs docker rmi || true
               pip install archerysec-cli
            '''
         }
      }
      stage('Code Security') {
         steps {
            parallel(
               dependency: {
                  withCredentials([usernamePassword(credentialsId: 'archerysec', passwordVariable: 'ARCHERYSEC_PASS', usernameVariable: 'ARCHERYSEC_USER')]) {
                     sh '''
                        docker run --env SECURE_LOG_LEVEL=${SECURE_LOG_LEVEL} -v "$PWD:/code" -v /var/run/docker.sock:/var/run/docker.sock registry.gitlab.com/gitlab-org/security-products/dependency-scanning:latest /code

                        # create project in archerysec

                        DATE=`date +%Y-%m-%d`

                        export PROJECT_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} --createproject \
                        --project_name=${PROJECT_NAME} --project_disc=${PROJECT_DISC} --project_start=${DATE} \
                        --project_end=${DATE} --project_owner=dev | tail -n1 | jq '.project_id' | sed -e 's/^"//' -e 's/"$//'`

                        # Upload Scan report in archerysec

                        export SCAN_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} \
                        --upload --file_type=JSON --file=${WORKSPACE}/gl-dependency-scanning-report.json --TARGET=${GIT_COMMIT} \
                        --scanner=gitlabsca --project_id=''$PROJECT_ID'' | tail -n1 | jq '.scan_id' | sed -e 's/^"//' -e 's/"$//'`

                        echo "Scan Report Uploaded Successfully, Scan Id:" $SCAN_ID
                     '''
                  }
               },
               sast: {
                  // echo 'SAST'
                  withCredentials([usernamePassword(credentialsId: 'archerysec', passwordVariable: 'ARCHERYSEC_PASS', usernameVariable: 'ARCHERYSEC_USER')]) {
                     sh '''
                        docker run --volume "$PWD":/code --volume /var/run/docker.sock:/var/run/docker.sock registry.gitlab.com/gitlab-org/security-products/sast:latest /app/bin/run /code

                        # create project in archerysec

                         DATE=`date +%Y-%m-%d`

                        export PROJECT_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} --createproject \
                        --project_name=${PROJECT_NAME} --project_disc=${PROJECT_DISC} --project_start=${DATE} \
                        --project_end=${DATE} --project_owner=dev | tail -n1 | jq '.project_id' | sed -e 's/^"//' -e 's/"$//'`

                        # Upload Scan report in archerysec

                        export SCAN_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} \
                        --upload --file_type=JSON --file=${WORKSPACE}/gl-sast-report.json --TARGET=${GIT_COMMIT} \
                        --scanner=gitlabsast --project_id=''$PROJECT_ID'' | tail -n1 | jq '.scan_id' | sed -e 's/^"//' -e 's/"$//'`

                        echo "Scan Report Uploaded Successfully, Scan Id:" $SCAN_ID
                      '''
                  }
               }
            )
         }
      }
      stage('Staging Setup') {
         steps {
            sh '''
               docker build  --no-cache --build-arg STAGE=staging -t "devops/webapp:staging" .
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
               withCredentials([usernamePassword(credentialsId: 'archerysec', passwordVariable: 'ARCHERYSEC_PASS', usernameVariable: 'ARCHERYSEC_USER')]) {
                  sh '''
                     # remove wrk folder
                     rm -rf wrk

                     # create wrk folder
                     mkdir wrk

                     chmod 777 wrk
                     docker run \
                        --volume $(pwd)/wrk:/output:rw \
                        --volume $(pwd)/wrk:/zap/wrk:rw \
                        registry.gitlab.com/gitlab-org/security-products/dast:latest /analyze -t ${TARGET_URL} -x report.xml

                     DATE=`date +%Y-%m-%d`

                     export PROJECT_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} --createproject \
                     --project_name=${PROJECT_NAME} --project_disc=${PROJECT_DISC} --project_start=${DATE} \
                     --project_end=${DATE} --project_owner=dev | tail -n1 | jq '.project_id' | sed -e 's/^"//' -e 's/"$//'`

                     # Upload Scan report in archerysec

                     export SCAN_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} \
                     --upload --file_type=XML --file=${WORKSPACE}/wrk/report.xml --TARGET=${GIT_COMMIT} \
                     --scanner=zap_scan --project_id=''$PROJECT_ID'' | tail -n1 | jq '.scan_id' | sed -e 's/^"//' -e 's/"$//'`

                     echo "Scan Report Uploaded Successfully, Scan Id:" $SCAN_ID
                  '''
               }
         }
      }
      stage('Production Setup') {
         steps {
            sh '''
               docker build  --no-cache --build-arg STAGE=prod -t "devops/webapp:prod" .
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
               withCredentials([usernamePassword(credentialsId: 'archerysec', passwordVariable: 'ARCHERYSEC_PASS', usernameVariable: 'ARCHERYSEC_USER')]) {
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

                     # create project in archerysec

                     DATE=`date +%Y-%m-%d`

                     export PROJECT_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} --createproject \
                     --project_name=${PROJECT_NAME} --project_disc=${PROJECT_DISC} --project_start=${DATE} \
                     --project_end=${DATE} --project_owner=dev | tail -n1 | jq '.project_id' | sed -e 's/^"//' -e 's/"$//'`

                     # Upload Scan report in archerysec

                     export SCAN_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} \
                     --upload --file_type=JSON --file=${WORKSPACE}/gl-container-scanning-report.json --TARGET=${GIT_COMMIT} \
                     --scanner=gitlabcontainerscan --project_id=''$PROJECT_ID'' | tail -n1 | jq '.scan_id' | sed -e 's/^"//' -e 's/"$//'`

                     echo "Scan Report Uploaded Successfully, Scan Id:" $SCAN_ID
                  '''
               }
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
    always {
      step([$class: 'WsCleanup'])
    }
  }
}