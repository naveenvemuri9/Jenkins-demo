#!/usr/bin/env groovy

pipeline {
    agent any 

    stages {
        stage('Checkout') {
            steps {
                       checkout scm
                       sh 'git clean -fdx'
                  }
        }
        stage('Build') { 
            steps { 
                sh 'pwd' 
            }
        }
        stage('Test'){
            steps {
                sh 'java -version'
                
            }
        }
        stage('Deploy') {
            steps {
                sh 'ls'
            }
        }
    }
}
