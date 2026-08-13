pipeline {
  agent {
    docker {
      image 'node:20-alpine'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }
  options {
    skipDefaultCheckout(true)
  }
  environment {
    DOCKER_REPO = 'sarandash2003-dotcom/ultimate-cicd'
  }
  stages {
    stage('Prepare Tooling') {
      steps {
        sh 'apk add --no-cache docker-cli git openjdk17-jre'
      }
    }
    stage('Checkout') {
      steps {
        sh 'rm -rf node-app/node_modules repo-temp || true'
        checkout scm
        sh 'echo "Starting build process..."'
      }
    }
    stage('Build and Test') {
      steps {
        sh '''
          cd node-app
          npm ci
          npm test
        '''
      }
    }
    
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonarqube') {
          sh '''
            cd node-app
            npx sonar-scanner \
              -Dsonar.projectKey=node-express-app \
              -Dsonar.projectName="Node Express App" \
              -Dsonar.sources=. \
              -Dsonar.exclusions=node_modules/**,coverage/** \
              -Dsonar.host.url=$SONAR_HOST_URL
          '''
        }
      }
    }

    stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "${DOCKER_REPO}:${BUILD_NUMBER}"
      }
      steps {
        script {
            sh 'docker build -t ${DOCKER_IMAGE} node-app'
            def dockerImage = docker.image("${DOCKER_IMAGE}")
            docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                dockerImage.push()
                dockerImage.push("latest")
            }
        }
      }
    }

    stage('Update Deployment File') {
      environment {
        GIT_REPO_NAME = "node-js-app-pipeline"
        GIT_USER_NAME = "dassaran504"
      }
      steps {
        withCredentials([
            string(
                credentialsId: 'github',
                variable: 'GITHUB_TOKEN'
            )
        ]) {
            sh '''
                rm -rf repo-temp
                git clone https://x-access-token:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git repo-temp
                cd repo-temp
                
                git config user.email "sarandash2003@gmail.com"
                git config user.name "${GIT_USER_NAME}"

                sed -i "s|image: .*|image: ${DOCKER_REPO}:${BUILD_NUMBER}|g" node-app-manifests/deployment.yml

                git add node-app-manifests/deployment.yml
                git commit -m "Update node app image tag to ${BUILD_NUMBER} [skip ci]" || echo "No changes to commit"
                git push https://x-access-token:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git main
            '''
        }
      }
    }
  }
  post {
    always {
      sh 'rm -rf node-app/node_modules repo-temp || true'
    }
  }
}
