pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }

  parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch')
    choice(name: 'DEPLOY_ENV', choices: ['staging','production'], description: 'Deploy env')
  }

  environment {
    GIT_URL = 'https://github.com/ihebmbarek1/spring-petclinic'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: "*/${params.BRANCH}"]],
          userRemoteConfigs: [[url: env.GIT_URL]]
        ])
        script {
          env.GIT_COMMIT_SHORT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
          env.BUILD_VERSION    = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
          env.DOCKER_IMAGE     = "spring-petclinic"
          env.DOCKER_TAG       = env.BUILD_VERSION
          echo "Commit=${env.GIT_COMMIT_SHORT}  BUILD_VERSION=${env.BUILD_VERSION}"
        }
      }
    }

    stage('Build') {
      steps {
        sh '''#!/bin/bash -leo pipefail
          set -e
          # Ensure SDKMAN and Java 25
          if [ ! -d "$HOME/.sdkman" ]; then
            curl -s "https://get.sdkman.io" | bash
          fi
          source "$HOME/.sdkman/bin/sdkman-init.sh"
          sdk install java 25-tem || true
          sdk use java 25-tem
          export JAVA_HOME="$HOME/.sdkman/candidates/java/current"
          export PATH="$JAVA_HOME/bin:$PATH"

          chmod +x mvnw || true
          ./mvnw -B -U -DskipTests=true -Dcheckstyle.skip=true clean package
        '''
      }
      post {
        always { archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, onlyIfSuccessful: false }
      }
    }

    stage('Parallel Testing') {
      parallel {
        stage('Unit Tests') {
          steps {
            sh '''#!/bin/bash -leo pipefail
              set -e
              source "$HOME/.sdkman/bin/sdkman-init.sh"
              sdk use java 25-tem
              export JAVA_HOME="$HOME/.sdkman/candidates/java/current"
              export PATH="$JAVA_HOME/bin:$PATH"

              ./mvnw -B -Dcheckstyle.skip=true \
                     -Dspring.docker.compose.skip.in-tests=true \
                     -Dtest=\\!PostgresIntegrationTests \
                     test
            '''
          }
          post {
            always { junit testResults: 'target/**/TEST-*.xml', allowEmptyResults: false }
          }
        }
        stage('Integration Tests (MySQL only)') {
          steps {
            sh '''#!/bin/bash -leo pipefail
              set -e
              source "$HOME/.sdkman/bin/sdkman-init.sh"
              sdk use java 25-tem
              export JAVA_HOME="$HOME/.sdkman/candidates/java/current"
              export PATH="$JAVA_HOME/bin:$PATH"

              ./mvnw -B -Dcheckstyle.skip=true \
                     -Dspring.docker.compose.skip.in-tests=true \
                     -Dtest=org.springframework.samples.petclinic.MySqlIntegrationTests \
                     verify
            '''
          }
          post {
            always { junit testResults: 'target/**/TEST-*.xml', allowEmptyResults: true }
          }
        }
      }
    }

    stage('Docker Image Build') {
      steps {
        sh '''
          docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
          docker images | head -n 5
        '''
      }
    }

    stage('Artifact Archiving') {
      steps {
        sh 'echo "image=${DOCKER_IMAGE}:${DOCKER_TAG}" > image.txt'
        archiveArtifacts artifacts: 'image.txt', fingerprint: true
      }
    }

    stage('Deployment (staging only)') {
      when { expression { params.DEPLOY_ENV == 'staging' && !env.CHANGE_ID } }
      steps {
        sh '''
          docker network inspect petnet >/dev/null 2>&1 || docker network create petnet
          docker rm -f petclinic-${BUILD_NUMBER} >/dev/null 2>&1 || true
          docker run -d --name petclinic-${BUILD_NUMBER} --network petnet -p 8082:8080 ${DOCKER_IMAGE}:${DOCKER_TAG}
          echo "Application deployed successfully."
        '''
      }
    }
  }

  post {
    success { echo "${env.JOB_NAME} #${env.BUILD_NUMBER}  ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}" }
    failure { echo " Build failed" }
    always  { archiveArtifacts artifacts: 'target/*.jar, image.txt', fingerprint: true, onlyIfSuccessful: false }
  }
}
