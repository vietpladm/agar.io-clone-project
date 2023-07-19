def imageTag = ''
pipeline {
  agent any
    environment {
    DOCKER_REGISTRY_USERNAME = credentials('DOCKER_REGISTRY_USERNAME')
    DOCKER_REGISTRY_PASSWORD = credentials('DOCKER_REGISTRY_PASSWORD')
    IMAGETAG = ""
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

    
    stage('Deploy to DEV environment') {
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
 
   stages {
    stage('Tag image of production version') {
      when {
        beforeInput true
        branch 'release'
      }
      steps {
        script {
          def userInput = input(
            message: "Enter release version... (example: v1.2.3)",
            parameters: [string(name: "IMAGE_TAG", defaultValue: "v0.0.0")]
          )
          imageTag = userInput.IMAGE_TAG
        }
        steps {
          sh '''
            echo "Tag image to release and push image"
            docker tag vietpl/agarioclone_agar:v2.${BUILD_NUMBER} vietpl/agarioclone_agar:${imageTag}
            echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
            docker push "vietpl/agarioclone_agar:${imageTag}"
          '''
        }
      }
    }
    
    stage('Update to the helm-chart of production') {
      when {
        beforeInput true
        branch 'release'
      }
      steps {
        script {
          sh "echo 'Update helm chart values'"
          def filename = 'argaio-helm/values.yaml'
          def data = readYaml file: filename
          data.image.tag = imageTag
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
            git commit -am "update image with new release tag as ${imageTag}"
            git push origin main
          '''
        }
      }
    }
  }
}