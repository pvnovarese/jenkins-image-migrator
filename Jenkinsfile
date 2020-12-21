//
// The included Jenkinsfile is a pipeline that will grab images from docker hub 
// (or some other public registry), retag them, push them into a scratch repo 
// on a private registry and queue them for scanning in Anchore. If the evaluation 
// passes, they will retag again and push into a "production" repo for internal 
// developers to use.
// 
// The hypothetical use case:
// 
// * internal developers are blocked from accessing Docker Hub (and other
//   public registries) by firewall rules
// * policy allows public images to be used if they pass a scan
// * the internal anchore and jenkins installations are behind the firewall
//   and cannot directly pull from Docker Hub (&c).
// * there is a jump host in the DMZ that we can ssh to and that can access 
//   external public registries, internal registries, and the Anchore API endpoint.

// this pipeline uses:
// anchore plugin for jenkins: https://www.jenkins.io/doc/pipeline/steps/anchore-container-scanner/
//

pipeline {
  environment {
    // probably don't need imageLine here
    // imageLine = 'pvnovarese/alpine-test:latest'
    SOURCE_IMAGE = 'busybox:latest'
    TARGET_REGISTRY = 'harbor-priv.novarese.net:443'
    TARGET_REPO = "${TARGET_REGISTRY}/hub-mirror/busybox"
    JUMP_HOST = 'anchore-priv.novarese.net' 
    SSH_ARGS = '-o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no'
    SCRATCH_IMAGE = "${TARGET_REPO}:${BUILD_NUMBER}-temp"
    PROD_IMAGE = "${TARGET_REPO}:${BUILD_NUMBER}-prod"
  }
  agent any

  stages {
    stage('Checkout SCM') {
      steps {
        checkout scm
      }
    }

    stage('Move Source Image to Scratch Repo') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'pvn-anchore-support.pem', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
          usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'HUB_USER', passwordVariable: 'HUB_PASS'),
          usernamePassword(credentialsId: 'harbor-priv.novarese.net', usernameVariable: 'TARGET_USER', passwordVariable: 'TARGET_PASS')
          ]) {
            sh '''
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker --version
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker login -u ${HUB_USER} -p ${HUB_PASS}
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker login -u ${TARGET_USER} -p ${TARGET_PASS} ${TARGET_REGISTRY}
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker pull ${SOURCE_IMAGE}
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker tag ${SOURCE_IMAGE} ${SCRATCH_IMAGE}
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker push ${SCRATCH_IMAGE}
              # once we've pushed it we don't need to keep the extra tag on the jump host
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker image rm ${SCRATCH_IMAGE}
              # just echoing here seems easier than using writeFile
              echo ${SCRATCH_IMAGE} > anchore_images
            '''          
          }      
      }
    }

    stage('Analyze with Anchore plugin') {
      steps {
        //don't need this because we wrote the anchore_images file in previous stage
        //writeFile file: 'anchore_images', text: imageLine

        // if we want to continue even if evaluation fails, use this line instead:
        // anchore name: 'anchore_images', bailOnFail: 'false'
        anchore name: 'anchore_images'
      }
    }

    stage('Push to Production Repo') {
      // we don't need to keep the target image once it's been pushed to the target repo
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'pvn-anchore-support.pem', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')
          ]) {
            sh '''
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker tag ${SOURCE_IMAGE} ${PROD_IMAGE}
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker push ${PROD_IMAGE}
              # once we've pushed it we don't need to keep the extra tag on the jump host
              ssh ${SSH_ARGS} -i ${SSH_KEY} ${SSH_USER}@${JUMP_HOST} docker image rm ${PROD_IMAGE} 
            '''
          }
      }
    }

  }
}




