//===-*- Groovy -*-===

pipeline {
    agent any
    environment {
        X86_JOB_NAME = ""
        TPU_MLIR_SHA = ""
        MODEL_ZOO_SHA = ""
    }
    stages {
        stage('Model-Zoo-Build') {
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
                                git clone --depth=1 file:////git-repository/tpu-mlir.git
                                ln -s /git-repository/model-zoo model-zoo
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
                        dir("$WORKSPACE/model-zoo") {
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
                                set -e
                                cd tpu-mlir/
                                source ./envsetup.sh
                                export PYTHONPATH=$WORKSPACE/python:\$PYTHONPATH
                                cd ../model-zoo/
                                python3 -m tpu_perf.build --mlir --list full_cases.txt || true
                            """
                            sh """
                                rm -f model-zoo
                                git clone /git-repository/model-zoo
                                mv -f /git-repository/model-zoo/output ./model-zoo/
                                cd model-zoo
                                chmod -R 777 ./output
                            """
                        }
                    }
                }
            }
        }
        stage('Test-Model-Zoo') {
            agent { label "bm1684x" }
            stages {
                stage("Model-Zoo-Test") {
                    steps {
                        dir("$WORKSPACE") {
                            sh """#!/bin/bash
                                set -e
                                rm -rf *
                                pip3 install https://github.com/sophgo/tpu-perf/releases/download/v1.1.5/tpu_perf-1.1.5-py3-none-manylinux2014_aarch64.whl --trusted-host mirrors.aliyun.com -i https://mirrors.aliyun.com/pypi/simple/
                                cd ../x86-agent-workspace/$X86_JOB_NAME/model-zoo
                                python3 -m tpu_perf.run --mlir --list full_cases.txt
                          """
                        }
                    }
                }
                stage("Save-Report-To-Database") {
                    environment {
                        PGDB_CREDS = credentials('postgres-user')
                    }
                    steps {
                        dir("$WORKSPACE") {
                            sh """#!/bin/bash
                               set -e

                               cd ../x86-agent-workspace/$X86_JOB_NAME/model-zoo

                               export PGPASSWORD=$PGDB_CREDS_PSW

                               psql -h 172.28.3.47 -p 8083 -U $PGDB_CREDS_USR -d tpu_mlir \\
                                    -c "\\COPY bm1684x_performance(name,shape,gops,\\"time(ms)\\",mac_utilization,cpu_usage,ddr_utilization) FROM './output/stats.csv' DELIMITER ',' CSV HEADER;"

                               psql -h 172.28.3.47 -p 8083 -U $PGDB_CREDS_USR -d tpu_mlir \\
                                    -c "UPDATE bm1684x_performance SET tpu_mlir_sha='$TPU_MLIR_SHA' WHERE tpu_mlir_sha IS NULL;"

                               psql -h 172.28.3.47 -p 8083 -U $PGDB_CREDS_USR -d tpu_mlir \\
                                    -c "UPDATE bm1684x_performance SET model_zoo_sha='$MODEL_ZOO_SHA' WHERE model_zoo_sha IS NULL;"

                               psql -h 172.28.3.47 -p 8083 -U $PGDB_CREDS_USR -d tpu_mlir \\
                                    -c "UPDATE bm1684x_performance SET date=CURRENT_TIMESTAMP WHERE date IS NULL;"
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
