pipeline {
  agent any
  environment {
    DOCKER_REGISTRY_USERNAME = credentials('DOCKER_REGISTRY_USERNAME')
    DOCKER_REGISTRY_PASSWORD = credentials('DOCKER_REGISTRY_PASSWORD')
  }

  stages {
    stage('Build') {
      steps {
        sh '''
          echo "Starting to build docker image"
          docker build -t vietpl/agarioclone_agar:v2.${BUILD_NUMBER} -f Dockerfile .
        '''
        sh '''
          echo "Starting to push docker image"
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
          docker push "vietpl/agarioclone_agar:v2.${BUILD_NUMBER}"
        '''
      }
    }
    stage('Deploy') {
      steps {
        script {
          sh "echo 'Deploy to kubernetes'"
          def filename = 'templates/deployment.yaml'
          def data = readYaml file: filename
          data.spec.template.spec.containers[0].image = "vietpl/agarioclone_agar:v2.${BUILD_NUMBER}"
          sh "rm $filename"
          writeYaml file: filename, data: data
          sh "cat $filename"
        }
        withKubeConfig([credentialsId: 'kubeconfig', serverUrl: 'https://172.21.161.250:6443']) {
          sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'  
          sh 'chmod u+x ./kubectl'  
          sh './kubectl apply -f templates'
        }
      }
    }
  }
}