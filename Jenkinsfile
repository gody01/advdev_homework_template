pipeline {
  agent any
  stages {
    stage('Prvi stage') {
      parallel {
        stage('Prvi stage') {
          steps {
            sh '''echo A
'''
          }
        }
        stage('Drugi stage') {
          steps {
            sh '''echo B
'''
          }
        }
      }
    }
    stage('Tretji stage') {
      steps {
        sh '''echo C
'''
      }
    }
  }
}