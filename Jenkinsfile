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
            wrap([$class: 'AnsiColorBuildWrapper', colorMapName: "xterm"]) {
                ansibleplaybook
                    colorized: true,
                    forks: 10,
                    inventory: 'inventory.ini',
                    limit: '',
                    playbook: 'postgres_rds.yml',
                    sudouser: null
 
            }
        }

    } catch(error) {
        throw error

    } finally {
        // Any cleanup operations needed, whether we hit an error or not

    }
}
