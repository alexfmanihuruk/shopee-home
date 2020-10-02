#!/usr/bin/env groovy

pipeline {
  agent { label 'master' }
  environment {
    tag_ver = sh (
      script: "git tag -l --sort=v:refname | tail -1",
      returnStdout: true
      ).trim()
      cluster_name = "${params.cluster_name}"
      project_name = "${params.project_name}"
      file_id = "${params.file_id}"
    }
    stages {
      stage('load env') {
        steps {
           configFileProvider([configFile(fileId: "${file_id}", variable: 'ENV_FILE')]) {
           echo " =========== ^^^^^^^^^^^^ Reading config from pipeline script "
           sh "rm -rf .env"
           sh "cp ${env.ENV_FILE} .env"
          }
        }
      }
      stage('build') {
        steps {
          sh "sed -i \'s:latest:${tag_ver}:\' docker-compose.yaml"
          sh "sed -i \'s:latest:${tag_ver}:\' .env"
          sh "docker-compose -f docker-compose.yaml build"
          sh "docker images"
        }
      }
      stage('cluster prepare') {
        steps {
          sh "kubectl create -n default --cluster gke_${project_name}_asia-southeast1-a_${cluster_name} secret generic shopee-home-secret --from-env-file=.env --dry-run -o yaml | kubectl apply -n default --cluster gke_${project_name}_asia-southeast1-a_${cluster_name} -f -"
          sh "sed -i \'s,shopee-home:latest,shopee-home:${tag_ver},g\' shopee-home.yaml"
        }
      }
      stage('push') {
        steps {
          retry(10) {
            sh "gcloud docker -- push gcr.io/${project_name}/shopee-home:${tag_ver}"
          }
          timeout(time: 20, unit: 'MINUTES') {
            sh 'echo checking health...'
          }
        }
      }
      stage('deploy') {
        steps {
          sh "kubectl apply -f shopee-home.yaml -n default --cluster gke_${project_name}_asia-southeast1-a_${cluster_name}"
        }
      }
    }
  }
