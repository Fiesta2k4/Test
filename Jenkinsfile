pipeline {
  agent any

  tools {
    maven 'Maven3' // tên Maven đã khai báo trong Jenkins: Manage Jenkins -> Global Tool Configuration
    jdk 'JDK17'    // tên JDK đã khai báo trong Jenkins
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B -DskipTests clean package'
      }
    }
  }

  post {
    success {
      archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
    always {
      junit 'target/surefire-reports/*.xml', allowEmptyResults: true
    }
  }
}