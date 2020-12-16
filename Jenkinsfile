// anchore plugin for jenkins: https://www.jenkins.io/doc/pipeline/steps/anchore-container-scanner/

pipeline {
  environment {
    // probably don't need imageLine here
    // imageLine = 'pvnovarese/alpine-test:latest'
    SOURCE_IMAGE = 'alpine:latest'
    targetRepo = 'pvnovarese/alpine-test'
    JUMP_HOST = 'anchore-priv.novarese.net'
    SSH_ARGS = '-o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no"'
  }
  agent any

  stages {
    stage('Checkout SCM') {
      steps {
        checkout scm
      }
    }

    stage('ssh to bastion host, pull source image and push to target repo') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'pvn-anchore-support.pem', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
          usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'HUB_USER', passwordVariable: 'HUB_PASS')
          ]) {
            sh '''
              ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker --version
              ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker login -u ${HUB_USER} -p ${HUB_PASS}
              ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker pull ${SOURCE_IMAGE}
              ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker tag ${SOURCE_IMAGE} ${targetRepo}:${BUILD_NUMBER}
              ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker push ${targetRepo}:${BUILD_NUMBER}
              # just echoing here seems easier than using writeFile
              echo ${SOURCE_IMAGE} > anchore_images
              echo ${targetRepo}:${BUILD_NUMBER} >> anchore_images 
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
      // if we succuessfully pushed the :prod tag than we don't need the $BUILD_NUMBER tag anymore
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'pvn-anchore-support.pem', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')
          ]) {
          sh 'ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker rmi  $targetRepo:${BUILD_NUMBER}'
          }
      }
    }

  }
}




