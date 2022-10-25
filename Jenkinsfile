pipeline {
   agent any
   environment {
     DOCKER_REGISTRY = "registry.devsecops:5000"
     SECURE_LOG_LEVEL = "debug"
     LOCAL_MACHINE_IP_ADDRESS="jenkins.devsecops"
     ARCHERYSEC_HOST ="http://archerysec.devsecops" // ArcherySec URL
     // These secrets should be use through Jenkins Secrets in production implementation
     ARCHERYSEC_USER = "admin" // archerysec username
     ARCHERYSEC_PASS = "devsecops@123A" // archerysec password

   }
   stages {
      stage('Build') {
         steps {
            sh '''
            # docker image and container clean up
            docker system prune -a -f
               docker images -f dangling=true -q | xargs docker rmi || true
            '''
         }
      }
      stage('Code Security') {
         steps {
            parallel(
               dependency: {
                  sh '''
                     docker run --env SECURE_LOG_LEVEL=${SECURE_LOG_LEVEL} -v "$PWD"/src:/code -v /var/run/docker.sock:/var/run/docker.sock registry.gitlab.com/gitlab-org/security-products/dependency-scanning:latest /code

                     # create project in archerysec

                     DATE=`date +%Y-%m-%d`

                     export PROJECT_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} --createproject \
                     --project_name=devsecops --project_disc="devsecops project" --project_start=${DATE} \
                     --project_end=${DATE} --project_owner=dev | tail -n1 | jq '.project_id' | sed -e 's/^"//' -e 's/"$//'`

                     # Upload Scan report in archerysec

                     export SCAN_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} \
                     --upload --file_type=JSON --file=${WORKSPACE}/gl-dependency-scanning-report.json --TARGET=${GIT_COMMIT} \
                     --scanner=gitlabsca --project_id=''$PROJECT_ID'' | tail -n1 | jq '.scan_id' | sed -e 's/^"//' -e 's/"$//'`

                     echo "Scan Report Uploaded Successfully, Scan Id:" $SCAN_ID
                  '''
               },
               sast: {
                  // echo 'SAST'
                  sh '''
                    docker run --volume "$PWD"/src:/code --volume /var/run/docker.sock:/var/run/docker.sock registry.gitlab.com/gitlab-org/security-products/sast:13-0-stable /app/bin/run /code

                     # create project in archerysec

                     DATE=`date +%Y-%m-%d`

                     export PROJECT_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} --createproject \
                     --project_name=devsecops --project_disc="devsecops project" --project_start=${DATE} \
                     --project_end=${DATE} --project_owner=dev | tail -n1 | jq '.project_id' | sed -e 's/^"//' -e 's/"$//'`

                     # Upload Scan report in archerysec

                     export SCAN_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} \
                     --upload --file_type=JSON --file=${WORKSPACE}/gl-sast-report.json --TARGET=${GIT_COMMIT} \
                     --scanner=gitlabsast --project_id=''$PROJECT_ID'' | tail -n1 | jq '.scan_id' | sed -e 's/^"//' -e 's/"$//'`

                     echo "Scan Report Uploaded Successfully, Scan Id:" $SCAN_ID
                  '''
               }
            )
         }
      }
      stage('Staging Setup') {
         steps {
            sh '''
               docker build  --no-cache --build-arg STAGE=staging -t "devsecops/webapp:staging" .
               docker tag "devsecops/webapp:staging" "${DOCKER_REGISTRY}/devsecops/webapp:staging"
               docker push "${DOCKER_REGISTRY}/devsecops/webapp:staging"
               docker rmi "${DOCKER_REGISTRY}/devsecops/webapp:staging"
               docker rmi "devsecops/webapp:staging"
            '''
            script {
                     def remote = [:]
                     remote.name = 'staging'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'staging.devsecops'
                     remote.identityFile = '~/.ssh/staging.key'
                     sshCommand remote: remote, command: "docker stop webapp || true"
                     sshCommand remote: remote, command: "docker rm webapp || true"
                     sshCommand remote: remote, command: "docker rmi ${DOCKER_REGISTRY}/devsecops/webapp:staging || true"
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
                remote.host = 'staging.devsecops'
                remote.identityFile = '~/.ssh/staging.key'
                sshCommand remote: remote, command: "docker run -d -p 80:5000 --name webapp ${DOCKER_REGISTRY}/devsecops/webapp:staging"
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
                  # remove wrk folder
                  rm -rf wrk

                  # create wrk folder
                  mkdir wrk

                  chmod 777 wrk
                  docker run \
                     --volume $(pwd)/wrk:/output:rw \
                     --volume $(pwd)/wrk:/zap/wrk:rw \
                     registry.gitlab.com/gitlab-org/security-products/dast:latest /analyze -t http://staging.devsecops -x report.xml

                  DATE=`date +%Y-%m-%d`

                 export PROJECT_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} --createproject \
                 --project_name=devsecops --project_disc="devsecops project" --project_start=${DATE} \
                 --project_end=${DATE} --project_owner=dev | tail -n1 | jq '.project_id' | sed -e 's/^"//' -e 's/"$//'`

                 # Upload Scan report in archerysec

                 export SCAN_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} \
                 --upload --file_type=XML --file=${WORKSPACE}/wrk/report.xml --TARGET=${GIT_COMMIT} \
                 --scanner=zap_scan --project_id=''$PROJECT_ID'' | tail -n1 | jq '.scan_id' | sed -e 's/^"//' -e 's/"$//'`

                 echo "Scan Report Uploaded Successfully, Scan Id:" $SCAN_ID
               '''
         }
      }
      stage('Production Setup') {
         steps {
            sh '''
               docker build  --no-cache --build-arg STAGE=prod -t "devsecops/webapp:prod" .
               docker tag "devsecops/webapp:prod" "${DOCKER_REGISTRY}/devsecops/webapp:prod"
               docker push "${DOCKER_REGISTRY}/devsecops/webapp:prod"
               docker rmi "${DOCKER_REGISTRY}/devsecops/webapp:prod"
               docker rmi "devsecops/webapp:prod"
            '''
            script {
                     def remote = [:]
                     remote.name = 'production'
                     remote.user = 'vagrant'
                     remote.allowAnyHosts = true
                     remote.host = 'production.devsecops'
                     remote.identityFile = '~/.ssh/production.key'
                     sshCommand remote: remote, command: "docker stop webapp || true"
                     sshCommand remote: remote, command: "docker rm webapp || true"
                     sshCommand remote: remote, command: "docker rmi ${DOCKER_REGISTRY}/devsecops/webapp:prod || true"
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
                  -e CI_APPLICATION_REPOSITORY=${DOCKER_REGISTRY}/devsecops/webapp \
                  -e CI_APPLICATION_TAG=prod \
                  -e REGISTRY_INSECURE=true \
                  registry.gitlab.com/gitlab-org/security-products/analyzers/klar

                  # create project in archerysec

                 DATE=`date +%Y-%m-%d`

                 export PROJECT_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} --createproject \
                 --project_name=devsecops --project_disc="devsecops project" --project_start=${DATE} \
                 --project_end=${DATE} --project_owner=dev | tail -n1 | jq '.project_id' | sed -e 's/^"//' -e 's/"$//'`

                 # Upload Scan report in archerysec

                 export SCAN_ID=`archerysec-cli -s ${ARCHERYSEC_HOST} -u ${ARCHERYSEC_USER} -p ${ARCHERYSEC_PASS} \
                 --upload --file_type=JSON --file=${WORKSPACE}/gl-container-scanning-report.json --TARGET=${GIT_COMMIT} \
                 --scanner=gitlabcontainerscan --project_id=''$PROJECT_ID'' | tail -n1 | jq '.scan_id' | sed -e 's/^"//' -e 's/"$//'`

                 echo "Scan Report Uploaded Successfully, Scan Id:" $SCAN_ID
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
                     remote.host = 'production.devsecops'
                     remote.identityFile = '~/.ssh/production.key'
                     sshCommand remote: remote, command: "docker run -d -p 80:5000 --name webapp ${DOCKER_REGISTRY}/devsecops/webapp:prod"
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