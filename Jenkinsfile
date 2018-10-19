def tag = 'UNKNOWN'
def hash = 'UNKNOWN'
def lambda_url = 'UNKNOWN'

pipeline {
  agent none
  stages {
    stage ('Get Code') {
      agent any
      steps {
        git 'https://github.com/Holmes89/hex-example.git'
      }
    }
    stage ('Test') {
      agent{
        docker {
            image 'golang:stretch'
            args '-e XDG_CACHE_HOME=/tmp/.cache'
        }
      }
      steps {
          script {
            sh 'go get -t ./...'
            sh 'go test ./...'
          }
      }
    }
    stage ('Build') {
      agent{
        docker {
            image 'golang:stretch'
            args '-e XDG_CACHE_HOME=/tmp/.cache'
        }
      }
      steps {
          script {
            sh 'go get ./...'
            sh 'GOOS=linux go build -o main main.go'
          }
      }
    }
    stage ('Container') {
        agent any
        steps {
          script {
            hash = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
            app = docker.build("holmes89/hex-example")
            docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                app.push("${hash}")
                app.push("latest")
            }
          }
        }
    }
    stage ('Tag Container') {
      when { buildingTag() }
      agent any
      steps {
        script {
          tag = sh(returnStdout: true, script: "git tag --sort version:refname | tail -1").trim()
          app = docker.build("holmes89/hex-example")
          docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
              app.push(tag)
          }
        }
      }
    }
    stage ('Deploy QA') {
        when { not { buildingTag() } }
        agent any
        steps {
          script {
            sh 'zip main.zip main'
            sh "aws s3 cp main.zip s3://hex-lambda/${hash}/main.zip"
            dir("terraform/qa"){
              sh 'terraform init'
              sh "terraform apply -var \"app_version=${hash}\" -auto-approve"
              lambda_url = sh(returnStdout: true, script: "terraform output url").trim()
            }
          }
        }
    }
    stage ('Prod Deploy') {
      when { buildingTag() }
      agent any
      steps {
        script {
          sh 'zip main.zip main'
          sh "aws s3 cp main.zip s3://hex-lambda/${tag}/main.zip"
          dir("terraform/prod"){
            sh 'terraform init'
            sh "terraform apply -var \"app_version=${tag}\" -auto-approve"
            lambda_url = sh(returnStdout: true, script: "terraform output url").trim()
          }
        }
      }
    }
    stage ('Acceptance Tests'){
      agent{
        docker {
            image 'python:3.7-alpine'
            args '-u root:sudo -e HOME=${env.WORKSPACE}'
        }
      }
      steps {
        script {
            dir("tests/acceptance"){
              sh '''source .env
                    robot -v HOST:${lambda_url}'''
            }
        }
      }
    }
  }
}
