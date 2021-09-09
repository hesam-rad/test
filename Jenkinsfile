#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'maven'
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
        stage('Deploy k8s') {
            environment {
                KUBE_URL = "https://168.119.234.58:6443"
            }
            steps {
                script {
                    echo "Deploy to the K8S ..."
                    withCredentials([string(credentialsId: 'kube-token', variable: 'KUBE_TOKEN')]) {
                        sh "kubectl config set-cluster k8s --server='${KUBE_URL}' --insecure-skip-tls-verify=true"
                        sh "kubectl config set-credentials admin --token='${KUBE_TOKEN}'"
                        sh "kubectl config set-context default --cluster=k8s --user=admin"
                        sh "kubectl config use-context default"
                        sh "envsubst < kubernetes/deployment.yaml | kubectl apply -f -"
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
                        sh 'git push origin HEAD:main'
                    }
                }
            }
        }
    }
}
