// anchore plugin for jenkins: https://www.jenkins.io/doc/pipeline/steps/anchore-container-scanner/

pipeline {
  environment {
    registry = 'registry.hub.docker.com'
    // you need a credential named 'docker-hub' with your DockerID/password to push images
    registryCredential = 'docker-hub'
    // change this repository and imageLine to your DockerID
    repository = 'pvnovarese/alpine-test'
    imageLine = 'pvnovarese/alpine-test:latest'
    SOURCE_IMAGE = 'alpine:latest'
    IMAGE_2 = 'pvnovarese/alpine-test:2020-12-16.${BUILD_ID}'
    targetRepo = 'pvnovarese/alpine-test'
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
              ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" -i ${SSH_KEY} ${SSH_USER}@anchore-priv.novarese.net docker --version
              ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" -i ${SSH_KEY} ${SSH_USER}@anchore-priv.novarese.net docker login -u ${HUB_USER} -p ${HUB_PASS}
              ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" -i ${SSH_KEY} ${SSH_USER}@anchore-priv.novarese.net docker pull ${SOURCE_IMAGE}
              ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" -i ${SSH_KEY} ${SSH_USER}@anchore-priv.novarese.net docker tag ${SOURCE_IMAGE} ${targetRepo}:${BUILD_NUMBER}
              ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" -i ${SSH_KEY} ${SSH_USER}@anchore-priv.novarese.net docker push ${targetRepo}:${BUILD_NUMBER}
              echo ${SOURCE_IMAGE} > anchore_images
              echo ${targetRepo}:${BUILD_NUMBER} >> anchore_images 
            '''          
          }      
      }
    }

    stage('Analyze with Anchore plugin') {
      steps {
        //writeFile file: 'anchore_images', text: imageLine
        anchore name: 'anchore_images', forceAnalyze: 'true', engineRetries: '900'
      }
    }

    stage('Clean up') {
      // if we succuessfully pushed the :prod tag than we don't need the $BUILD_ID tag anymore
      steps {
        sh 'docker rmi $targetRepo:${BUILD_NUMBER}'
      }
    }

  }
}




