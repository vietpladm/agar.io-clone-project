pipeline {
  agent any
    environment {
    DOCKER_REGISTRY_USERNAME = credentials('DOCKER_REGISTRY_USERNAME')
    DOCKER_REGISTRY_PASSWORD = credentials('DOCKER_REGISTRY_PASSWORD')
  }


  // Stage ví dụ để nhánh bugfix và feature sẽ thực thi quá trình test ở CI.
  stages {
    stage('Install dependencies and Test') {
      steps {
        echo "pseudo install dependencies"
        sh '''
          echo "pseudo run test case"
          echo "pseudo run integration test"
          echo "pseudo check code quality"
        '''
      }
    }

    // Stage build Docker image, chỉ áp dụng cho  2 nhánh develop và release
    stage('Build Docker Image') {
      when {
        anyOf {
          branch 'develop'
          branch 'release'
        }
      }
      steps {
        sh '''
          echo "Starting to build docker image"
          docker build -t vietpl/agarioclone_agar:v2.${BUILD_NUMBER} -f Dockerfile .
        '''
      }
    }

   // Với pipeline của nhánh develop, push docker image lên Docker Hub

    stage('Create & Tag image & Update Helm-Chart for DEV Environment') {
  when {
    branch 'develop'
  }
  steps {
    // Tag image and push to Docker Hub
    sh '''
      echo "Tag image to release and push image to Docker Hub"
      docker tag vietpl/agarioclone_agar:v2.${BUILD_NUMBER} vietpl/agarioclone_agar:${BUILD_NUMBER}-dev
      echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
      docker push "vietpl/agarioclone_agar:${BUILD_NUMBER}-dev"
    '''

    // Update Helm-Chart values for DEV Environment
    script {
      withCredentials([gitUsernamePassword(credentialsId: 'jenkins_github_pac', gitToolName: 'Default')]) {
        sh 'rm -rf argaio-helm-dev'
        sh 'git clone https://github.com/vietpladm/argaio-helm-dev.git'
      }
      sh "echo 'Update Helm chart values'"
      def filename = 'argaio-helm-dev/values.yaml'
      def data = readYaml file: filename
      data.image.tag = "${BUILD_NUMBER}-dev"
      sh "rm $filename"
      writeYaml file: filename, data: data
      sh "cat $filename"
    }
    withCredentials([gitUsernamePassword(credentialsId: 'jenkins_github_pac', gitToolName: 'Default')]) {
      sh '''
        cd argaio-helm-dev
        git config user.email "phan1@chie.cf"
        git config user.name "vietpladm"
        git add values.yaml
        git commit -am "Update image with new dev tag as ${BUILD_NUMBER}-dev"
        git push origin main
      '''
    }
  }
}


    // Nếu là nhánh release, yêu cầu nhập vào version cho ứng dụng để đánh tag và update Helm-chart repo chuẩn bị cho ArgoCD step.
 
stage('Tag image and update Helm-Chart for PRD') {
  when {
    beforeInput true
    branch 'release'
  }
  input {
    message "Enter release version ... (example: v1.2.3) to release to production environment"
    ok "Confirm"
    parameters {
      string(name: "IMAGE_TAG", defaultValue: "v0.0.0")
    }
  }
  steps {
    sh '''
      echo "Tag image to release and push image to docker hub"
      docker tag vietpl/agarioclone_agar:v2.${BUILD_NUMBER} vietpl/agarioclone_agar:${IMAGE_TAG}
      echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
      docker push "vietpl/agarioclone_agar:${IMAGE_TAG}"
    '''
    script {
      withCredentials([gitUsernamePassword(credentialsId: 'jenkins_github_pac', gitToolName: 'Default')]) {
        sh 'rm -rf argaio-helm'
        sh 'git clone https://github.com/vietpladm/argaio-helm.git'
      }
      sh "echo 'Update helm chart values'"
      def filename = 'argaio-helm/values.yaml'
      def data = readYaml file: filename
      data.image.tag = IMAGE_TAG
      sh "rm $filename"
      writeYaml file: filename, data: data
      sh "cat $filename"
    }
    withCredentials([gitUsernamePassword(credentialsId: 'jenkins_github_pac', gitToolName: 'Default')]) {
      sh '''
        cd argaio-helm
        git config user.email "phan1@chie.cf"
        git config user.name "vietpladm"
        git add values.yaml
        git commit -am "update image with new release tag as ${IMAGE_TAG}"
        git push origin main
      '''
    }
  }
 }
}
}