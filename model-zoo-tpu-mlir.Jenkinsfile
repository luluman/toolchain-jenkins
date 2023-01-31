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
                script {
                    X86_WORKSPACE = WORKSPACE
                }
                dir("$WORKSPACE") {
                    sh """#!/bin/bash
                        set -e
                        cd tpu-mlir/
                        source ./envsetup.sh
                        cd ../
                        mkdir python -p
                        pip install https://github.com/sophgo/tpu-perf/releases/download/v1.1.5/tpu_perf-1.1.5-py3-none-manylinux2014_x86_64.whl --trusted-host mirrors.aliyun.com -i https://mirrors.aliyun.com/pypi/simple/ -t python
                        export PYTHONPATH=$WORKSPACE/python:\$PYTHONPATH
                        cd model-zoo/
                        python3 -m tpu_perf.build --mlir --list full_cases.txt
                        mv -rf output ../
                    """
                }
            }
        }
        stage('Test-Model-Zoo') {
            agent { label "bm1684x" }
            steps {
                dir("$WORKSPACE") {
                    echo "$X86_WORKSPACE"
                    sh """#!/bin/bash
                        set -e
                        pip install https://github.com/sophgo/tpu-perf/releases/download/v1.1.5/tpu_perf-1.1.5-py3-none-manylinux2014_x86_64.whl --trusted-host mirrors.aliyun.com -i https://mirrors.aliyun.com/pypi/simple/ --user
                        python3 -m tpu_perf.build --help
                    """
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
