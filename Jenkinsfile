pipeline {
  agent any
  stages {
    stage('Setup') {
      steps {
        sh 'echo "Hello World setup"'
      }
    }
    stage('Test') {
      agent {
        docker {
          image 'python:3.5.1'
        }
        
      }
      steps {
        sh 'python hello.py'
      }
    }
    stage('Deploy') {
      steps {
        sh 'echo "Deploy somewhere..."'
      }
    }
  }
}