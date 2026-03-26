pipeline {
  agent any

  tools {
    jdk 'JDK17'
    maven 'Maven3'
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Build artifact') {
      steps {
        sh 'mvn -B -DskipTests clean package'
      }
    }
  }

  post {
    success {
      archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
  }
}