//===-*- Groovy -*-===

def generateStage(model) {
    return {
        stage("${model}") {
                dir("$WORKSPACE") {
                    sh """#!/bin/bash
                        set -e
                        cd tpu-mlir/
                        source ./envsetup.sh
                        cd regression
                        ./run_model.sh ${model}
                        """
            }
        }
    }
}

pipeline {
    parameters {
        string(name: 'COMMIT_SHA', defaultValue: '', description: 'Build the specific version of TPU-MLIR .')
        text(name: 'CHECK_MODELS', defaultValue: '', description: 'Models should be rechecked under this version. Separated by "\\n"')
        booleanParam(name: 'FAIL_FAST', defaultValue: false, description: 'Control whether we should exit early when one job in parallel is failed.')
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
                        ln -s /git-repository/nnmodels nnmodels
                        cd tpu-mlir
                        git checkout ${params.COMMIT_SHA} -q
                        git show -s
                    """
                }
            }
        }
        stage('Build') {
            steps {
                dir("$WORKSPACE") {
                    sh """#!/bin/bash
                        set -e
                        cd tpu-mlir/
                        source ./envsetup.sh
                        ./build.sh
                    """
                }
            }
        }
        stage('Test') {
            steps {
                script {
                    if (params.CHECK_MODELS?.trim()) {
                        String[] models = params.CHECK_MODELS.split("\n")
                        def parallelStagesMap = models.collectEntries {
                            ["${it}" : generateStage(it)]
                        }
                        parallelStagesMap["failFast"] = params.FAIL_FAST
                        parallel parallelStagesMap
                    } else {
                        dir("$WORKSPACE") {
                                sh """#!/bin/bash
                                set -e
                                cd tpu-mlir/
                                source ./envsetup.sh
                                cd regression
                                ./run_all.sh
                            """
                        }
                    }
                }
            }
        }
    }
}
