#!/usr/bin/env groovy

import org.jenkinsci.plugins.workflow.steps.FlowInterruptedException

def compressBrewLogs() {
    echo "Compressing brew logs.."
    commonlib.shell(script: "./find-and-compress-brew-logs.sh")
}

def isMassRebuild() {
    return currentBuild.displayName.contains("[mass rebuild]")
}

node {
    checkout scm
    def buildlib = load("pipeline-scripts/buildlib.groovy")
    def commonlib = buildlib.commonlib
    def slacklib = commonlib.slacklib

    commonlib.describeJob("k_ocp4", """
        Konflux
    """)


    // Expose properties for a parameterized build
    properties(
        [
            disableResume(),
            buildDiscarder(
                logRotator(
                    artifactDaysToKeepStr: '30',
                    daysToKeepStr: '30')),
            [
                $class: 'ParametersDefinitionProperty',
                parameterDefinitions: [
                    commonlib.dryrunParam(),
                    commonlib.mockParam(),
                    commonlib.artToolsParam(),
                    booleanParam(
                        name: 'IGNORE_LOCKS',
                        description: 'Do not wait for other builds in this version to complete (use only if you know they will not conflict)',
                        defaultValue: false
                    ),
                    commonlib.ocpVersionParam('BUILD_VERSION', '4'),
                    string(
                        name: 'NEW_VERSION',
                        description: '(Optional) version for build instead of most recent\nor "+" to bump most recent version',
                        defaultValue: "",
                        trim: true,
                    ),
                    string(
                        name: 'ASSEMBLY',
                        description: 'The name of an assembly to rebase & build for. If assemblies are not enabled in group.yml, this parameter will be ignored',
                        defaultValue: "test",
                        trim: true,
                    ),
                    string(
                        name: 'DOOZER_DATA_PATH',
                        description: 'ocp-build-data fork to use (e.g. test customizations on your own fork)',
                        defaultValue: "https://github.com/openshift-eng/ocp-build-data",
                        trim: true,
                    ),
                    string(
                        name: 'DOOZER_DATA_GITREF',
                        description: '(Optional) Doozer data path git [branch / tag / sha] to use',
                        defaultValue: "",
                        trim: true,
                    ),
                    string(
                        name: 'IMAGE_LIST',
                        description: '(Optional) Comma/space-separated list to include/exclude per BUILD_IMAGES (e.g. logging-kibana5,openshift-jenkins-2)',
                        defaultValue: "",
                        trim: true,
                    ),
                ]
            ],
        ]
    )

    commonlib.checkMock()

    if (currentBuild.description == null) {
        currentBuild.description = ""
    }
    sshagent(["openshift-bot"]) {
        stage("ocp4") {
            // artcd command
            def cmd = [
                "artcd",
                "-v",
                "--working-dir=./artcd_working",
                "--config=./config/artcd.toml",
            ]
            if (params.DRY_RUN) {
                cmd << "--dry-run"
            }
            cmd += [
                "k_ocp4",
                "--version=${params.BUILD_VERSION}",
                "--assembly=${params.ASSEMBLY}",
            ]
            if (params.DOOZER_DATA_PATH) {
                cmd << "--data-path=${params.DOOZER_DATA_PATH}"
            }
            if (params.DOOZER_DATA_GITREF) {
                cmd << "--data-gitref=${params.DOOZER_DATA_GITREF}"
            }
            cmd += [
                "--image-list=${commonlib.cleanCommaList(params.IMAGE_LIST)}"
            ]

            buildlib.withAppCiAsArtPublish() {
                withCredentials([
                            string(credentialsId: 'jenkins-service-account', variable: 'JENKINS_SERVICE_ACCOUNT'),
                            string(credentialsId: 'jenkins-service-account-token', variable: 'JENKINS_SERVICE_ACCOUNT_TOKEN'),
                            string(credentialsId: 'redis-server-password', variable: 'REDIS_SERVER_PASSWORD'),
                            string(credentialsId: 'redis-host', variable: 'REDIS_HOST'),
                            string(credentialsId: 'redis-port', variable: 'REDIS_PORT'),
                            string(credentialsId: 'gitlab-ocp-release-schedule-schedule', variable: 'GITLAB_TOKEN'),
                            string(credentialsId: 'openshift-bot-token', variable: 'GITHUB_TOKEN'),
                            string(credentialsId: 'jboss-jira-token', variable: 'JIRA_TOKEN'),
                            aws(credentialsId: 's3-art-srv-enterprise', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'),
                            string(credentialsId: 'art-bot-slack-token', variable: 'SLACK_BOT_TOKEN'),
                            usernamePassword(credentialsId: 'art-dash-db-login', passwordVariable: 'DOOZER_DB_PASSWORD', usernameVariable: 'DOOZER_DB_USER'),
                            file(credentialsId: 'art-jenkins-ldap-serviceaccount-private-key', variable: 'RHSM_PULP_KEY'),
                            file(credentialsId: 'art-jenkins-ldap-serviceaccount-client-cert', variable: 'RHSM_PULP_CERT'),
                        ]) {
                    withEnv(['DOOZER_DB_NAME=art_dash']) {
                        sh "rm -rf ./artcd_working && mkdir -p ./artcd_working"
                        sh(script: cmd.join(' '), returnStdout: true)
                    }
                }
            }
        }
    }
}
