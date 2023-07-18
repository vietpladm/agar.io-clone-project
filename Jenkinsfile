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
    stage('Push Docker Image in develop') {
      when {
        branch 'develop'
      }
      // Đánh tag dev cho image
      steps {
        sh '''
          echo "Tag image to dev and push image"
          docker tag vietpl/agarioclone_agar:v2.${BUILD_NUMBER} vietpl/agarioclone_agar:v2.${BUILD_NUMBER}-dev
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
          docker push "vietpl/agarioclone_agar:v2.${BUILD_NUMBER}-dev"
        '''
      }
    }

    // Deploy tới môi trường development, tương ứng là namespace dev trên Kubernetes
    stage('Deploy to development environment') {
      when {
        branch 'develop'
      }
      steps {
        script {
          sh "echo 'Deploy to kubernetes'"
          echo "Update tag in deployment"
          def filename = 'templates/deployment.yaml'
          def data = readYaml file: filename
          data.spec.template.spec.containers[0].image = "vietpl/agarioclone_agar:v2.${BUILD_NUMBER}-dev"
          sh "rm $filename"
          writeYaml file: filename, data: data
          sh "cat $filename"
          echo "Update service Node Port"
          def servicefile = 'templates/service.yaml'
          def servicedata = readYaml file: servicefile
          servicedata.spec.ports[0].nodePort = 30190
          sh "rm $servicefile"
          writeYaml file: servicefile, data: servicedata
        }
        withKubeConfig([credentialsId: 'kubeconfig', serverUrl: 'https://172.21.161.250:6443']) {
          sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'
          sh 'chmod u+x ./kubectl'
          sh './kubectl apply -f templates -n dev'
        }
      }
    }

    // Nếu là nhánh release, yêu cầu nhập vào version cho ứng dụng để đánh tag và triển khai.
    stage('Tag image of production version') {
      when {
        beforeInput true
        branch 'release'
      }
      // Yêu cầu nhập vào tag cho release image
      input {
        message "Enter release version... (example: v1.2.3)"
        ok "Confirm"
        parameters {
          string(name: "IMAGE_TAG", defaultValue: "v0.0.0")
        }
      }
      steps {
        sh '''
          echo "Tag image to release and push image"
          docker tag vietpl/agarioclone_agar:v2.${BUILD_NUMBER} vietpl/agarioclone_agar:${IMAGE_TAG}
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
          docker push "vietpl/agarioclone_agar:${IMAGE_TAG}"
        '''
        
      }
    }
    stage('Update to the helm-chart of production') {

       // Confirmation to update helm-chart of production?
      input {
        message "Approve to deploy to ArgoCD PRD?"
        ok "Confirm"
      }

      steps {
        withCredentials([gitUsernamePassword(credentialsId: 'jenkins_github_pac', gitToolName: 'Default')]) {
          sh 'rm -rf argaio-helm'
          sh 'git clone https://github.com/vietpladm/argaio-helm.git'
        }
        script {
          sh "echo 'Update helm chart values'"
          def filename = 'argaio-helm/values.yaml'
          def data = readYaml file: filename
          data.image.tag = "${IMAGE_TAG}"
          sh "rm $filename"
          writeYaml file: filename, data: data
          sh "cat $filename"
        }
        withCredentials([gitUsernamePassword(credentialsId: 'jenkins_github_pac', gitToolName: 'Default')]) {
          sh '''
            cd argaio-helm
            git config user.email "jenkins@example.com"
            git config user.name "Jenkins"
            git add values.yaml
            git commit -am "update image with new release tag as ${IMAGE_TAG}"
            git push origin main
          '''
        }
      }
    }
  }
}