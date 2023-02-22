//===-*- Groovy -*-===

pipeline {
    agent {
        docker {
            image 'sophgo/tpuc_dev'
            label 'local'
            // args '-v /home/toolchain/jenkins/workspace:/workspace'
        }
    }
    stages {
        stage('Setup') {
            steps {
                build job: 'sync-gerrit'
                dir("$WORKSPACE") {
                    sh """#!/bin/bash
                        set -e
                        rm -rf *
                        git clone --depth=20 file:////git-repository/tpu-mlir.git
                        ln -s /git-repository/nnmodels nnmodels
                    """
                }
            }
        }
        stage('Build') {
            steps {
                dir("$WORKSPACE/tpu-mlir") {
                    sh """#!/bin/bash
                        set -e
                        source ./envsetup.sh
                        ./build.sh
                    """
                }
            }
        }
        stage('Test') {
            // options {
            //     timeout(time: 4, unit: 'HOURS')   // timeout on this stage
            // }
            steps {
                dir("$WORKSPACE/tpu-mlir") {
                    sh """#!/bin/bash
                        set -e
                        source ./envsetup.sh
                        cd regression
                        ./run_all.sh
                    """
                }
            }
            post {
                failure {
                    dir("$WORKSPACE/tpu-mlir") {
                        script {
                            def result_log = readFile "regression/regression_out/result.log"
                            def failed_cases = result_log =~/(.+)\ +regression\ +FAILED/
                            def failed = []
                            for (item in failed_cases) {
                                failed.add(item[1])
                            }
                            // https://github.com/jenkinsci/pipeline-plugin/blob/master/TUTORIAL.md#serializing-local-variables
                            failed_cases = null
                            def failed_models = failed.join("\n")
                            def commit_20_sha = sh(script: '''git log  --pretty=oneline -n 20 --pretty=format:\"%h\"''', returnStdout: true)
                            def commit_20 = commit_20_sha.split("\n")
                            def first_bad = commit_20[0]
                            def commit_19 = commit_20[1..-1]
                            for (commit in commit_19) {
                                def b = build job: 'blame-commit-tpu-mlir', parameters: [
                                        string(name: 'COMMIT_SHA', value: commit),
                                        text(name: 'CHECK_MODELS', value: failed_models),
                                        booleanParam(name: 'FAIL_FAST', value: true)
                                        ], wait: true, propagate: false
                                def varb = b.getResult()
                                if (varb == "SUCCESS") {
                                    build job: 'email-bad-commit-tpu-mlir', parameters: [
                                        string(name: 'COMMIT_SHA', value: first_bad),
                                        text(name: 'CHECK_MODELS', value: failed_models)
                                    ], wait: false, propagate: false
                                    error("Bad commit introduced in:\n ${first_bad}.")
                                } else {
                                    first_bad = commit
                                }
                            }
                        }
                    }
                }
            }
        }
        stage("Release") {
            steps {
                dir("$WORKSPACE/tpu-mlir") {
                    sh """#!/bin/bash
                        set -e
                        git clean -ffdx
                        ./release.sh
                    """
                }
            }
        }
    }
}
