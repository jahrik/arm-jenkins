#!/usr/bin/env groovy

docker_login = 'docker-login'
docker_email = 'docker-email'

node('master') {

    try {

        stage('build') {
            deleteDir()
            checkout scm
        }

        stage('test') {
        }

        stage('deploy') {
          withCredentials([
            usernamePassword(credentialsId: docker_login,
              usernameVariable: 'DOCKER_USER',
              passwordVariable: 'DOCKER_PASS'),
            string(credentialsId: docker_email,
              variable: 'DOCKER_EMAIL')]) {
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
      deleteDir()
    }

}
