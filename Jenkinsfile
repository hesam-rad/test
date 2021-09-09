#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'maven'
    }
    triggers {
        pollSCM('* * * * *')
    }
    stages {
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage('build app') {
            steps {
                script {
                    echo "building the application..."
                    sh 'mvn clean package'
                }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'docker-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build -t docker.ehsan.cf/demo-app:${IMAGE_NAME} ."
                        sh "echo $PASS | docker login docker.ehsan.cf -u $USER --password-stdin"
                        sh "docker push docker.ehsan.cf/demo-app:${IMAGE_NAME}"
                    }
                }
            }
        }
        stage('commit version update') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh 'git config --global user.email "admin@gmail.com"'
                        sh 'git config --global user.name "admin"'
                        sh "git remote set-url origin https://${USER}:${PASS}@github.com/hesam-rad/test.git/"
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump"'
                        sh 'git push origin HEAD:master'
                    }
                }
            }
        }
    }
}
