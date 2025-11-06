pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')                   // requires AnsiColor plugin
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
    skipDefaultCheckout(true)            // avoid the implicit "Declarative: Checkout SCM"
  }

  triggers { pollSCM('@daily') }

  environment {
    APP_NAME         = 'spring-petclinic'
    DOCKER_NAMESPACE = 'ihebmb'          // your Docker Hub namespace
    DOCKER_IMAGE     = "${env.DOCKER_NAMESPACE}/${env.APP_NAME}"
    DOCKER_CREDS_ID  = 'dockerhub-creds' // Jenkins creds id (user/pass to Docker Hub)
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Compute Version') {
      steps {
        script {
          // Read Maven POM safely (no undefined 'project' var)
          def pom = readMavenPom file: 'pom.xml'
          env.APP_NAME     = pom.artifactId ?: env.APP_NAME
          env.POM_VERSION  = pom.version    ?: '0.0.0'

          // Short git SHA + build number
          env.GIT_SHORT    = sh(returnStdout: true, script: 'git rev-parse --short=8 HEAD').trim()
          env.APP_VERSION  = "${env.POM_VERSION}-${env.BUILD_NUMBER}-${env.GIT_SHORT}"

          echo "APP_NAME=${env.APP_NAME}"
          echo "POM_VERSION=${env.POM_VERSION}"
          echo "APP_VERSION=${env.APP_VERSION}"
        }
        // Persist the calculated version into the built jar name
        sh "./mvnw -B versions:set -DnewVersion=${APP_VERSION} -DgenerateBackupPoms=false"
      }
    }

    stage('Tests (parallel)') {
      parallel {
        stage('Unit') {
          steps {
            sh "./mvnw -B -DskipITs=true test"
            junit allowEmptyResults: false, testResults: 'target/surefire-reports/*.xml'
          }
        }
        stage('Integration') {
          steps {
            // PetClinic may not have ITs; run verify anyway and publish if present
            sh "./mvnw -B verify"
            junit allowEmptyResults: true, testResults: 'target/failsafe-reports/*.xml'
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
        echo "Application deployed successfully."
      }
    }
  }

  post {
    success {
      script {
        try {
          emailext to: 'ihebmbarek738@gmail.com',
            subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Version ${env.APP_VERSION} built and pushed as ${env.DOCKER_IMAGE}:${env.APP_VERSION}\n${env.BUILD_URL}"
        } catch (e) {
          echo "Email skipped: ${e.message}"
        }
      }
    }
    failure {
      script {
        try {
          emailext to: 'ihebmbarek738@gmail.com',
            subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Build failed. ${env.BUILD_URL}"
        } catch (e) {
          echo "Email skipped: ${e.message}"
        }
      }
    }
  }
}
