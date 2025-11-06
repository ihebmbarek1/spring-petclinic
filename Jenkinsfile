pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
  }

  triggers { pollSCM('@daily') } 

  environment {
    APP_NAME         = 'spring-petclinic'
    DOCKER_NAMESPACE = 'ihebmb'  
    DOCKER_IMAGE     = "${env.DOCKER_NAMESPACE}/${env.APP_NAME}"
    DOCKER_CREDS_ID  = 'dockerhub-creds'
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Compute Version') {
      steps {
        script {
          def sha = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          def base = sh(script: "./mvnw -q -Dexec.executable=echo -Dexec.args='\\${project.version}' --non-recursive org.codehaus.mojo:exec-maven-plugin:3.1.0:exec", returnStdout: true).trim()
          env.APP_VERSION = "${base}-${env.BUILD_NUMBER}-${sha}"
        }
        sh "./mvnw -B versions:set -DnewVersion=${APP_VERSION} -DgenerateBackupPoms=false"
      }
    }

    stage('Tests (parallel)') {
      parallel {
        stage('Unit') {
          steps {
            sh "./mvnw -B -DskipITs=true test"
            junit 'target/surefire-reports/*.xml'
          }
        }
        stage('Integration') {
          steps {
            sh "./mvnw -B -P integration-tests verify"
            junit 'target/failsafe-reports/*.xml'
          }
        }
      }
    }

    stage('Package') {
      steps {
        sh "./mvnw -B -DskipTests package"
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }

    stage('Docker Build & Push') {
      steps {
        script {
          docker.withRegistry('', DOCKER_CREDS_ID) {
            def img = docker.build("${DOCKER_IMAGE}:${APP_VERSION}")
            img.push()
            img.push('latest')
          }
        }
      }
    }

    stage('Deploy (main only)') {
      when { branch 'main' }
      steps {
        sh """
          docker rm -f ${APP_NAME} || true
          docker run -d --name ${APP_NAME} -p 8080:8080 ${DOCKER_IMAGE}:${APP_VERSION}
        """
      }
    }
  }

  post {
    success {
      emailext to: 'ihebmbarek738@gmail.com',
        subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: "Version ${env.APP_VERSION} built and pushed as ${env.DOCKER_IMAGE}:${env.APP_VERSION}\n${env.BUILD_URL}"
    }
    failure {
      emailext to: 'ihebmbarek738@gmail.com',
        subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: "Build failed. ${env.BUILD_URL}"
    }
  }
}
