#!/usr/bin/env groovy

/* IMPORTANT:
 *
 * In order to make this pipeline work, the following configuration on Jenkins is required:
 * - slave with a specific label (see pipeline.agent.label below)
 * - credentials plugin should be installed and have the secrets with the following names:
 *   + lciadm100credentials (token to access Artifactory)
 */

def defaultBobImage = 'armdocker.rnd.ericsson.se/proj-adp-cicd-drop/bob.2.0:2.2.6'
def bob = new BobCommand()
        .bobImage(defaultBobImage)
        .envVars([ISO_VERSION: '${ISO_VERSION}'])
        .needDockerSocket(true)
        .toString()
def GIT_COMMITTER_NAME = 'enmadm100'
def GIT_COMMITTER_EMAIL = 'enmadm100@ericsson.com'
def failedStage = ''

pipeline {
    agent {
        label 'Cloud-Native'
    }
    stages {
        stage('Init') {
            steps {
                withCredentials([file(credentialsId: 'lciadm100-docker-auth', variable: 'dockerConfig')]) {
                    sh "install -m 600 ${dockerConfig} ${HOME}/.docker/config.json"
                }
                git branch: 'master',
                        url: 'ssh://gerrit.ericsson.se:29418/OSS/ENM-Parent/SQ-Gate/com.ericsson.oss.containerisation/eric-enm-common-ruleset'
            }
        }
        stage('Generate version') {
            steps {
                sh "${bob} generate-new-version"
            }
            post {
                failure {
                    script {
                        failedStage = env.STAGE_NAME
                    }
                }
            }
        }
        stage('Retrieve version') {
            steps {
                script {
                    env.RULESET_VERSION = sh(script: "cat .bob/var.version", returnStdout:true).trim()
                    echo "${RULESET_VERSION}"
                    sh "echo 'New Common Ruleset Version : ${RULESET_VERSION}' > artifact.properties"
                }
                archiveArtifacts 'artifact.properties'
            }
            post {
                failure {
                    script {
                        failedStage = env.STAGE_NAME
                    }
                }
            }
        }
        stage('Publish') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'cenmbuild_ARM_token', usernameVariable: 'CENMBUILD_USERNAME', passwordVariable: 'CENMBUILD_ACCESS_TOKEN')]) {
                        def bobWithHelmToken = new BobCommand()
                            .bobImage(defaultBobImage)
                            .needDockerSocket(true)
                            .envVars([
                                'CENMBUILD_ACCESS_TOKEN': env.CENMBUILD_ACCESS_TOKEN,
                                'ARM_USER': env.CENMBUILD_USERNAME,
                                'ARM_COMMON_RULESET_REPO': "proj-eric-oss-drop-generic"
                            ])
                            .toString()
                        sh "${bobWithHelmToken} publish"
                    }
                }
            }
            post {
                failure {
                    script {
                        failedStage = env.STAGE_NAME
                    }
                }
            }
        }
        stage('Tag Git Repository') {
            steps {
                wrap([$class: 'BuildUser']) {
                    script {
                        def bobWithCommitterInfo = new BobCommand()
                                .bobImage(defaultBobImage)
                                .needDockerSocket(true)
                                .envVars([
                                        'AUTHOR_NAME'        : "\${BUILD_USER:-${GIT_COMMITTER_NAME}}",
                                        'AUTHOR_EMAIL'       : "\${BUILD_USER_EMAIL:-${GIT_COMMITTER_EMAIL}}",
                                        'GIT_COMMITTER_NAME' : "${GIT_COMMITTER_NAME}",
                                        'GIT_COMMITTER_EMAIL': "${GIT_COMMITTER_EMAIL}"
                                ])
                                .toString()
                        sh "${bobWithCommitterInfo} create-git-tag"
                    }
                }
            }
            post {
                failure {
                    script {
                        failedStage = env.STAGE_NAME
                    }
                }
                always {
                    script {
                        sh "${bob} remove-git-tag"
                    }
                }
            }
        }
        stage('Bump Version') {
            steps {
                script {
                    sh '''
                        echo ${WORKSPACE}
                        chmod -R 777 ${WORKSPACE}
                    '''
                    sh 'hostname'
                    Version = readFile "VERSION_PREFIX"
                    sh 'docker run --rm -v $PWD/VERSION_PREFIX:/app/VERSION -w /app armdocker.rnd.ericsson.se/proj-enm/bump patch'
                    newVersion = readFile "VERSION_PREFIX"
                    env.IMAGE_VERSION = newVersion
                    currentBuild.displayName = "${BUILD_NUMBER} - Version - " + Version
                    sh '''
                        git add VERSION_PREFIX
                        git commit -m "Version $IMAGE_VERSION"
                        git push origin HEAD:master
                    '''
                }
            }
        }
    }
    post {
        failure {
            mail to: 'PDLCHAKRAS@pdl.internal.ericsson.com',
                    subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
                    body: "Failure on ${env.BUILD_URL}"
        }
    }
}

// More about @Builder: http://mrhaki.blogspot.com/2014/05/groovy-goodness-use-builder-ast.html
import groovy.transform.builder.Builder
import groovy.transform.builder.SimpleStrategy

@Builder(builderStrategy = SimpleStrategy, prefix = '')
class BobCommand {
    def bobImage = 'bob.2.0:latest'
    def envVars = [:]
    def needDockerSocket = false

    String toString() {
        def env = envVars
                .collect({ entry -> "-e ${entry.key}=\"${entry.value}\"" })
                .join(' ')

        def cmd = """\
            |docker run
            |--init
            |--rm
            |--workdir \${PWD}
            |--user \$(id -u):\$(id -g)
            |-v \${PWD}:\${PWD}
            |-v /etc/group:/etc/group:ro
            |-v /etc/passwd:/etc/passwd:ro
            |-v \${HOME}/.m2:\${HOME}/.m2
            |-v \${HOME}/.docker:\${HOME}/.docker
            |${needDockerSocket ? '-v /var/run/docker.sock:/var/run/docker.sock' : ''}
            |${env}
            |\$(for group in \$(id -G); do printf ' --group-add %s' "\$group"; done)
            |--group-add \$(stat -c '%g' /var/run/docker.sock)
            |${bobImage}
            |"""
        return cmd
                .stripMargin()           // remove indentation
                .replace('\n', ' ')      // join lines
                .replaceAll(/[ ]+/, ' ') // replace multiple spaces by one
    }
}
