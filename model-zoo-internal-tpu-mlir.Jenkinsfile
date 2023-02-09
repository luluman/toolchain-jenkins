//===-*- Groovy -*-===

pipeline {
    agent any
    environment {
        X86_JOB_NAME = ""
        TPU_MLIR_SHA = ""
        MODEL_ZOO_SHA = ""
    }
    stages {
        stage('Model-Zoo-Internal-Build') {
            agent {
                docker {
                    image 'sophgo/tpuc_dev'
                    label 'local'
                }
            }
            stages {
                stage('Checkout') {
                    steps {
                        dir("$WORKSPACE") {
                            sh """#!/bin/bash
                                set -e
                                rm -rf *
                                git clone --depth=1 --branch=release file:////git-repository/tpu-mlir.git
                                ln -s /git-repository/model-zoo-internal model-zoo-internal
                                cd tpu-mlir
                                git show -s
                            """
                        }
                        dir("$WORKSPACE/tpu-mlir") {
                            script {
                                X86_JOB_NAME = WORKSPACE.split('/')[-1]
                                TPU_MLIR_SHA = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                            }
                        }
                        dir("$WORKSPACE/model-zoo-internal") {
                            script {
                                MODEL_ZOO_SHA = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                            }
                        }
                    }
                }
                stage('Build-TPU-MLIR') {
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
                stage('Build-Model-Zoo-Internal') {
                    steps {
                        dir("$WORKSPACE") {
                            sh """#!/bin/bash
                                set -e
                                cd tpu-mlir/
                                source ./envsetup.sh
                                cd ../
                                mkdir python -p
                                pip install https://github.com/sophgo/tpu-perf/releases/download/v1.1.5/tpu_perf-1.1.5-py3-none-manylinux2014_x86_64.whl --trusted-host mirrors.aliyun.com -i https://mirrors.aliyun.com/pypi/simple/ -t python
                            """
                            sh """#!/bin/bash
                                set -e
                                cd tpu-mlir/
                                source ./envsetup.sh
                                export PYTHONPATH=$WORKSPACE/python:\$PYTHONPATH
                                cd ../model-zoo-internal/
                                python3 -m tpu_perf.build --mlir --list full_cases.txt || true
                                find ./output -name "*.npz" -type f -delete
                            """
                            sh """
                                rm -f model-zoo-internal
                                git clone /git-repository/model-zoo-internal
                                mv -f /git-repository/model-zoo-internal/output ./model-zoo-internal/
                                cd model-zoo-internal
                                chmod -R 777 ./output
                            """
                        }
                    }
                }
            }
        }
        stage('Test-Model-Zoo-Internal') {
            agent { label "bm1684x" }
            stages {
                stage("Model-Zoo-Internal-Test") {
                    steps {
                        dir("$WORKSPACE") {
                            sh """#!/bin/bash
                                set -e
                                rm -rf *
                                pip3 install https://github.com/sophgo/tpu-perf/releases/download/v1.1.5/tpu_perf-1.1.5-py3-none-manylinux2014_aarch64.whl --trusted-host mirrors.aliyun.com -i https://mirrors.aliyun.com/pypi/simple/
                                cd ../x86-agent-workspace/$X86_JOB_NAME/model-zoo-internal
                                python3 -m tpu_perf.run --mlir --list full_cases.txt
                          """
                        }
                    }
                }
            }
        }
    }
    // post {
    //     cleanup {
    //         dir("$WORKSPACE") {
    //             sh "rm -rf *"
    //         }
    //     }
    // }
}
