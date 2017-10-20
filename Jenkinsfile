
#!groovy 
//def "naveen=application name"
def archive="naveen-svr-${env.BRANCH_NAME.replaceAll(/[^A-Za-z0-9\-]/,"_")}-${env.BUILD_NUMBER}.zip"
properties([
         [$class: 'GithubProjectProperty', 
         displayName: '', 
         projectUrlStr: 'https://github.com/ModusCreateOrg/GORUCK-app-server/'], 
         [$class: 'RebuildSettings', autoRebuild: false, rebuildDisabled: false], 
         pipelineTriggers([upstream('feature/separate-data_fetcher'), 
         githubPush()])
])

def notifyStarted() {
  // send to Slack
  slackSend (color: '#FFFF00', message: "STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
  // send to email
  emailext (
      subject: "STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
      body: """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
        <p>Check console output at "<a href="${env.BUILD_URL}">${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>"</p>""",
      recipientProviders: [[$class: 'DevelopersRecipientProvider']]
    )
}
def notifySuccessful() {
  slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})") 
  emailext (
      subject: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
      body: """<p>SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
        <p>Check console output at "<a href="${env.BUILD_URL}">${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>"</p>""",
      recipientProviders: [[$class: 'DevelopersRecipientProvider']]
    )
}
def notifyFailed() {
   slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
   emailext (
       subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
       body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
         <p>Check console output at "<a href="${env.BUILD_URL}">${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>"</p>""",
       recipientProviders: [[$class: 'DevelopersRecipientProvider']]

     )
 }

try {
    notifyStarted()
    pipeline {
        agent {
            label 'ubuntu-14.04-amd64'
        }
        parameters {
            choice(name: 'ENVIRONMENT', description: 'Used for posgress.sh and CodeDeploy deployment group.', choices: 'integration\nstaging\nprod')
            booleanParam(name: 'REBUILD_DATABASE', defaultValue: false, description: 'Should we rebuild the database?') 
            booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Should we deploy the application?') 
        }

        def deploymentGroupName = params.ENVIRONMENT ?: 'integration'
        def shouldDeploy = params.DEPLOY ?: false
        def shouldRebuildDatabase = params.REBUILD_DATABASE ?: false

        stages {
            stage('Checkout') {
                steps {
                    checkout scm
                    sh 'git clean -fdx'
                }
            }
            stage('Build') {
                steps {
                    nodejs('NodeJS 4.2.1') {
                        dir ('svr') {
                            sh """#!/usr/bin/env bash
                                  set -x
                                  set -euo pipefail
                                  npm install --${deploymentGroupName}"""
                            }
                        }
                }
            }
            if (shouldRebuildDatabase) {
                stage('Rebuild Database') {
                    steps {
                        nodejs('NodeJS 4.2.1') {
                            dir ('svr') {
                                sh """#!/usr/bin/env bash
                                      set -x
                                      set -euo pipefail
                                      set -a
                                      tmpfile="\$(mktemp)"
                                      aws s3 cp "s3://naveen-codedeploy/web/${deploymentGroupName}.env" "\$tmpfile"
                                      source "\$tmpfile"
                                      node ./initdb_postgres.js --debug
                                      rm -f "\$tmpfile"
                                   """
                            }
                        }
                    }
                }
            }
            if (shouldDeploy) {
                stage('Deploy') {
                    steps {
                         echo 'Deploying...'
                         sh """#!/usr/bin/env bash
                            set -x
                            set -euo pipefail
                            mkdir -p build
                            rsync -a bin svr appspec.yml build/
                            # If we have built the database, we don't want to archive
                            # those build products, so force remove this directory
                            rm -rf svr/sql
                            cd build
                            aws deploy push \
                                --application-name naveen \
                                --s3-location s3://naveen-codedeploy/web/${archive} \
                                --region us-east-1
                            aws deploy create-deployment \
                                --region us-east-1 \
                                --application-name naveen \
                                --deployment-group-name ${deploymentGroupName} \
                                --s3-location bucket=Name-codedeploy,bundleType=zip,key=web/${archive}"""
          
                        }
                    } 
                }
            }
        }
    notifySuccessful()
} catch (e) {
    notifyFailed()
    currentBuild.result = "FAILED"
    throw e
  }
