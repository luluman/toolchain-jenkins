//===-*- Groovy -*-===

pipeline {
    parameters {
        string(name: 'COMMIT_SHA', defaultValue: '', description: 'The commit of TPU-MLIR .')
        text(name: 'CHECK_MODELS', defaultValue: '', description: 'Models should be rechecked under this version. Separated by "\\n"')
    }
    agent {
        docker {
            image 'sophgo/tpuc_dev'
            label 'local'
        }
    }
    stages {
        stage('Information') {
            steps {
                echo "TPU-MLIR version: ${params.COMMIT_SHA}\nModels:\n${params.CHECK_MODELS}"
            }
        }
        stage('Checkout') {
            steps {
                dir("$WORKSPACE") {
                    sh """#!/bin/bash
                        set -e
                        rm -rf *
                        git clone file:////git-repository/tpu-mlir.git
                        cd tpu-mlir
                        git checkout ${params.COMMIT_SHA} -q
                        git show -s
                    """
                }
            }
        }
        stage('Test') {
            steps {
                script {
                    dir("$WORKSPACE/tpu-mlir") {
                        script {
                            def b = build job: 'blame-commit-tpu-mlir', parameters: [
                                string(name: 'COMMIT_SHA', value: params.COMMIT_SHA),
                                text(name: 'CHECK_MODELS', value: params.CHECK_MODELS),
                                booleanParam(name: 'FAIL_FAST', value: false)
                                ], wait: true, propagate: false
                            def commit_msg = sh(script: "git show -s ${params.COMMIT_SHA}", returnStdout: true)
                            def author_email = sh(script: "git show --format=%ae -s ${params.COMMIT_SHA}", returnStdout: true)
                            def buildURL = b.getAbsoluteUrl()
                            def jobName = b.getProjectName()
                            def blueBuildURL = buildURL.replace("job/${jobName}", "blue/organizations/jenkins/${jobName}/detail/${jobName}")
                            emailext subject: 'TPU-MLIR daily-test failed',
                                to: author_email,
                                body: """
TPU-MLIR failed to pass some tests due to this commit:

${commit_msg}

----------------------------------

Affected cases:
${params.CHECK_MODELS}

----------------------------------

More info at: ${blueBuildURL}"""
                        }
                    }
                }
            }
        }
    }
    post {
        cleanup {
            dir("$WORKSPACE") {
                sh "rm -rf *"
            }
        }
    }
}
