#!/usr/bin/env groovy

docker_login = '0ae69f01-26c0-427d-a5f0-d1ad65a18b62'
env.DOCKER_EMAIL = 'jahrik@gmail.com'

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
          withCredentials([usernamePassword(credentialsId: docker_login,
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS')]) {
            echo "Running ${env.BUILD_ID} on ${env.JENKINS_URL}"
            echo "DOCKER_USER = ${env.DOCKER_USER}"
            echo "DOCKER_PASS = ${env.DOCKER_PASS}"
            echo "DOCKER_EMAIL = ${env.DOCKER_EMAIL}"
            ansiColor('xterm') {
                ansiblePlaybook(
                    playbook: 'playbook.yml',
                    inventory: 'inventory.ini',
                    // limit: 'local',
                    colorized: true)
            }
          }
        }

    } catch(error) {
        throw error

    } finally {
        // Any cleanup operations needed, whether we hit an error or not

    }

}
