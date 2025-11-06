pipeline {
  agent any

  tools {
    jdk 'temurin-17'           // Configure in Jenkins Global Tools as "temurin-17"
  }

  options {
    timestamps()
    ansiColor('xterm')
    timeout(time: 20, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20'))
  }

  parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch to build')
    choice(name: 'DEPLOY_ENV', choices: ['staging','production'], description: 'Deployment environment')
  }

  environment {
    GIT_URL        = 'https://github.com/ihebmbarek1/spring-petclinic'
    IMAGE_NAME     = 'ihebmbarek1/spring-petclinic'   // change to your Docker Hub repo: <user>/<repo>
    REGISTRY       = 'https://index.docker.io/v1/'
    DOCKER_CREDS   = 'dockerhub-creds'                // Jenkins credentials id
    GIT_CREDS      = 'github-creds'                   // Jenkins credentials id (optional if public)
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: "*/${params.BRANCH}"]],
          userRemoteConfigs: [[url: env.GIT_URL, credentialsId: env.GIT_CREDS]]
        ])
        script {
          env.GIT_COMMIT_SHORT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
          env.BUILD_VERSION    = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
          env.BRANCH_SAFE      = sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD | tr "/" "-"').trim()
          echo "Commit=${env.GIT_COMMIT_SHORT}  BUILD_VERSION=${env.BUILD_VERSION}  BRANCH=${env.BRANCH_SAFE}"
        }
      }
    }

    stage('Parallel Testing') {
      parallel {
        stage('Unit Tests') {
          steps {
            sh '''
              ./mvnw -B -U -Dspring.docker.compose.skip.in-tests=true test
            '''
          }
          post {
            always {
              junit testResults: 'target/surefire-reports/*.xml, target/failsafe-reports/*.xml', allowEmptyResults: false
            }
          }
        }
        stage('Integration Tests (MySQL only)') {
          steps {
            // Prefer a Maven profile in pom.xml; this is a fallback.
            sh '''
              ./mvnw -B -Dspring.docker.compose.skip.in-tests=true \
                     -Dtest=org.springframework.samples.petclinic.MySqlIntegrationTests \
                     verify
            '''
          }
          post {
            always {
              junit testResults: 'target/surefire-reports/*.xml, target/failsafe-reports/*.xml', allowEmptyResults: true
            }
          }
        }
      }
    }

    stage('Package (JAR)') {
      steps {
        sh '''
          chmod +x mvnw || true
          ./mvnw -B -DskipTests=true clean package
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, onlyIfSuccessful: false
        }
      }
    }

    stage('Docker Build & Push') {
      steps {
        script {
          // Tag matrix: build-version, latest, branch
          env.IMG_TAG_BUILD  = env.BUILD_VERSION
          env.IMG_TAG_LATEST = 'latest'
          env.IMG_TAG_BRANCH = env.BRANCH_SAFE

          sh """
            docker build -t ${IMAGE_NAME}:${IMG_TAG_BUILD} .
            docker tag ${IMAGE_NAME}:${IMG_TAG_BUILD} ${IMAGE_NAME}:${IMG_TAG_LATEST}
            docker tag ${IMAGE_NAME}:${IMG_TAG_BUILD} ${IMAGE_NAME}:${IMG_TAG_BRANCH}
          """

          docker.withRegistry(env.REGISTRY, env.DOCKER_CREDS) {
            sh """
              docker push ${IMAGE_NAME}:${IMG_TAG_BUILD}
              docker push ${IMAGE_NAME}:${IMG_TAG_LATEST}
              docker push ${IMAGE_NAME}:${IMG_TAG_BRANCH}
            """
          }
        }
      }
    }

    stage('Artifact Marker') {
      steps {
        sh '''
          echo "image_build=${IMAGE_NAME}:${IMG_TAG_BUILD}"   > image.txt
          echo "image_latest=${IMAGE_NAME}:${IMG_TAG_LATEST}" >> image.txt
          echo "image_branch=${IMAGE_NAME}:${IMG_TAG_BRANCH}" >> image.txt
        '''
        archiveArtifacts artifacts: 'image.txt', fingerprint: true
      }
    }

    stage('Deploy: Staging (single host)') {
      when { expression { params.DEPLOY_ENV == 'staging' && !env.CHANGE_ID } }
      steps {
        sh '''
          docker network inspect petnet >/dev/null 2>&1 || docker network create petnet
          # Stop and remove any previous instance
          docker rm -f petclinic-staging >/dev/null 2>&1 || true
          # Run container (host 8082 -> container 8080)
          docker run -d --name petclinic-staging --network petnet -p 8082:8080 \
            --health-cmd="wget -qO- http://localhost:8080/actuator/health || exit 1" \
            --health-interval=20s --health-timeout=5s --health-retries=10 \
            ${IMAGE_NAME}:${IMG_TAG_BUILD}

          echo "Waiting for health..."
          for i in $(seq 1 30); do
            status=$(docker inspect --format='{{json .State.Health.Status}}' petclinic-staging || echo "null")
            if [ "$status" = "\"healthy\"" ]; then echo "Application healthy."; exit 0; fi
            sleep 2
          done
          echo "Application not healthy in time"; docker logs --tail=200 petclinic-staging; exit 1
        '''
      }
    }
  }

  post {
    success {
      echo "✅ ${env.JOB_NAME} #${env.BUILD_NUMBER} → ${env.IMAGE_NAME}:${env.IMG_TAG_BUILD}"
    }
    failure {
      echo "❌ Build failed"
    }
    always {
      archiveArtifacts artifacts: 'target/*.jar, image.txt', fingerprint: true, onlyIfSuccessful: false
    }
  }
}
