// anchore plugin for jenkins: https://www.jenkins.io/doc/pipeline/steps/anchore-container-scanner/

pipeline {
  environment {
    // probably don't need imageLine here
    // imageLine = 'pvnovarese/alpine-test:latest'
    SOURCE_IMAGE = 'alpine:latest'
    TARGET_REPO = 'pvnovarese/alpine-test'
    JUMP_HOST = 'anchore-priv.novarese.net'
    SSH_ARGS = '-o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no'
  }
  agent any

  stages {
    stage('Checkout SCM') {
      steps {
        checkout scm
      }
    }

    stage('ssh to bastion host, pull source image and push to temporary repo') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'pvn-anchore-support.pem', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
          usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'HUB_USER', passwordVariable: 'HUB_PASS')
          ]) {
            sh '''
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker --version
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker login -u ${HUB_USER} -p ${HUB_PASS}
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker pull ${SOURCE_IMAGE}
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker tag ${SOURCE_IMAGE} ${TARGET_REPO}:temp-${BUILD_NUMBER}
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker push ${TARGET_REPO}:temp-${BUILD_NUMBER}
              # once we've pushed it we don't need to keep the extra tag on the jump host
              # ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker rmi  ${TARGET_REPO}:temp-${BUILD_NUMBER}
              # just echoing here seems easier than using writeFile
              echo ${SOURCE_IMAGE} > anchore_images
              echo ${TARGET_REPO}:temp-${BUILD_NUMBER} >> anchore_images 
            '''          
          }      
      }
    }

    stage('Analyze with Anchore plugin') {
      steps {
        //writeFile file: 'anchore_images', text: imageLine
        anchore name: 'anchore_images', bailOnFail: 'false'
      }
    }

    stage('Clean up') {
      // we don't need to keep the target image once it's been pushed to the target repo
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'pvn-anchore-support.pem', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')
          ]) {
            sh '''
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker tag ${SOURCE_IMAGE} ${TARGET_REPO}:${BUILD_NUMBER}-prod
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker push ${TARGET_REPO}:${BUILD_NUMBER}-prod
              # once we've pushed it we don't need to keep the extra tag on the jump host
              # ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker rmi  ${TARGET_REPO}:${BUILD_NUMBER}-prod
            '''
          }
      }
    }

  }
}




