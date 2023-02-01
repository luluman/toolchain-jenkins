//===-*- Groovy -*-===

pipeline {
    agent {
        docker {
            image 'sophgo/tpuc_dev'
            label 'local'
        }
    }
    environment {
        X86_WORKSPACE = ""
        TPU_MLIR_SHA = ""
        MODEL_ZOO_SHA = ""
        TASK_DATA = ""
    }
    stages {
        stage('Checkout') {
            steps {
                dir("$WORKSPACE") {
                    sh """#!/bin/bash
                        set -e
                        rm -rf *
                        git clone --depth=1 file:////git-repository/tpu-mlir.git
                        ln -s /git-repository/model-zoo model-zoo
                        cd tpu-mlir
                        git show -s
                    """
                }
                dir("$WORKSPACE/tpu-mlir") {
                    script {
                        X86_WORKSPACE = WORKSPACE
                        TPU_MLIR_SHA = sh(script: "git rev-parse HEAD", returnStdout: true)
                    }
                }
                dir("$WORKSPACE/model-zoo") {
                    script {
                        MODEL_ZOO_SHA = sh(script: "git rev-parse HEAD", returnStdout: true)
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
        stage('Build-Model-Zoo') {
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
                        set +e
                        cd tpu-mlir/
                        source ./envsetup.sh
                        export PYTHONPATH=$WORKSPACE/python:\$PYTHONPATH
                        cd ../model-zoo/
                        python3 -m tpu_perf.build --mlir --list full_cases.txt
                    """
                    sh """
                        rm model-zoo
                        git clone /git-repository/model-zoo
                        mv -f /git-repository/model-zoo/output ./model-zoo/
                        cd mode-zoo
                        chmod -R 777 ./output
                    """
                }
            }
        }
        stage('Test-Model-Zoo') {
            agent { label "bm1684x" }
            steps {
                script {
                    def job_name = X86_WORKSPACE.split('/')[-1]
                    dir("$WORKSPACE") {
                        echo "$X86_WORKSPACE"
                        sh """#!/bin/bash
                            set -e
                            pip install https://github.com/sophgo/tpu-perf/releases/download/v1.1.5/tpu_perf-1.1.5-py3-none-manylinux2014_x86_64.whl --trusted-host mirrors.aliyun.com -i https://mirrors.aliyun.com/pypi/simple/
                            cd $job_name
                            python3 -m tpu_perf.run --mlir --list full_cases.txt
                      """
                    }
                }
            }
        }
    }
    // post {
        // cleanup {
        //     dir("$WORKSPACE") {
        //         sh "rm -rf *"
        //     }
        // }
    // }
}
