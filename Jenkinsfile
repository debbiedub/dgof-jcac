pipeline {
  agent { docker {
      image 'python:3'
    }
  }
  stages {
    stage('Test') {
      steps {
        sh "git help"
      }
    }
  }
}
