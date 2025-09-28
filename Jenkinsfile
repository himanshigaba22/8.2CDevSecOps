pipeline {
  agent any

  tools { nodejs 'NodeJS' }

  environment {
    APP_NAME = 'your-app'
    DOCKER_IMAGE = "himanshi29/${APP_NAME}"
    SONARQUBE_SERVER = 'SonarQubeCloud'
    DOCKERHUB_CREDS = 'dockerhub-creds'
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/himanshigaba22/8.2CDevSecOps.git'
      }
    }

    stage('Build') {
      steps {
        sh 'npm ci'
        sh 'npm run build || echo "no build step"'
        sh 'docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .'
      }
    }

    stage('Test') {
      steps {
        sh 'npm test -- --ci --coverage || true'
      }
      post {
        always { junit allowEmptyResults: true, testResults: '**/junit.xml' }
      }
    }

    stage('Code Quality: SonarCloud') {
      steps {
        withSonarQubeEnv(SONARQUBE_SERVER) {
          withEnv(["PATH+SCAN=${tool 'SonarScanner'}/bin"]) {
            sh 'sonar-scanner'
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true }
      }
    }

    stage('Security') {
      steps {
        sh 'npm audit --audit-level=high || true'
        sh '''
          if ! command -v trivy >/dev/null; then
            curl -sSfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
          fi
          trivy image --no-progress --exit-code 0 --severity CRITICAL,HIGH ${DOCKER_IMAGE}:${BUILD_NUMBER}
          trivy image --no-progress --exit-code 1 --severity CRITICAL ${DOCKER_IMAGE}:${BUILD_NUMBER} || echo "CRITICAL vulns found; review report"
        '''
      }
    }

    stage('Deploy: Staging') {
      steps {
        withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDS, usernameVariable: 'USER', passwordVariable: 'PASS')]) {
          sh 'echo "$PASS" | docker login -u "$USER" --password-stdin'
          sh 'docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}'
        }
        sh "BUILD_NUMBER=${BUILD_NUMBER} docker compose -f docker-compose.staging.yml up -d --force-recreate"
