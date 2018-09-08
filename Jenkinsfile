#!/usr/bin/env groovy

node('master') {

    try {

        stage('build') {
            // Clean workspace
            deleteDir()
            // Checkout the app at the given commit sha from the webhook
            checkout scm
        }

        stage('test') {
            // Run any testing suites
            sh "echo 'WE ARE TESTING'"
        }

        stage('deploy') {
            sh "echo 'WE ARE DEPLOYING'"
            ansiColor('xterm') {
                ansiblePlaybook(
                    playbook: 'playbook.yml',
                    inventory: 'inventory.ini',
                    colorized: true)
            }
        }

    } catch(error) {
        throw error

    } finally {
        // Any cleanup operations needed, whether we hit an error or not

    }
}
