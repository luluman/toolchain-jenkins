//===-*- Groovy -*-===

pipeline {
    parameters {
        string(name: 'TPU_MLIR_BRANCH', defaultValue: 'master', description: 'The branch of TPU-MLIR.')
        string(name: 'MODEL_ZOO_REPO', defaultValue: 'model-zoo', description: 'The repository of Model-Zoo.')
        string(name: 'MODEL_ZOO_BRANCH', defaultValue: 'main', description: 'The branch of Model-Zoo.')
        booleanParam(name: 'COMMIT_TO_DATASET', defaultValue: false, description: 'Whether commit report to database.')
    }
    agent any
    environment {
        X86_JOB_NAME = ""
        TPU_MLIR_SHA = ""
        MODEL_ZOO_SHA = ""
    }
    stages {
        stage("${params.MODEL_ZOO_REPO}-Build") {
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
                                git clone --depth=1 --brach=${params.TPU_MLIR_BRANCH} file:////git-repository/tpu-mlir.git
                                ln -s /git-repository/${params.MODEL_ZOO_REPO} ${params.MODEL_ZOO_REPO}
                                cd ${params.MODEL_ZOO_REPO}
                                git lfs checkout ${params.MODEL_ZOO_BRANCH}
                                git show -s
                                cd ../${params.MODEL_ZOO_REPO}
                                git show -s
                            """
                        }
                        dir("$WORKSPACE/tpu-mlir") {
                            script {
                                X86_JOB_NAME = WORKSPACE.split('/')[-1]
                                TPU_MLIR_SHA = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                            }
                        }
                        dir("$WORKSPACE/$params.MODEL_ZOO_REPO") {
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
                stage("Build-${params.MODEL_ZOO_REPO}") {
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
                                cd ../${params.MODEL_ZOO_REPO}/
                                python3 -m tpu_perf.build --mlir --list full_cases.txt || true
                                find ./output -name "*.npz" -type f -delete
                            """
                            sh """
                                rm -f ${params.MODEL_ZOO_REPO}
                                git clone /git-repository/${params.MODEL_ZOO_REPO}
                                mv -f /git-repository/${params.MODEL_ZOO_REPO}/output ./${params.MODEL_ZOO_REPO}/
                                cd ${params.MODEL_ZOO_REPO}
                                chmod -R 777 ./output
                            """
                        }
                    }
                }
            }
        }
        stage("Test-${params.MODEL_ZOO_REPO}") {
            agent { label "bm1684x" }
            stages {
                stage("${params.MODEL_ZOO_REPO}-Test") {
                    steps {
                        dir("$WORKSPACE") {
                            sh """#!/bin/bash
                                set -e
                                rm -rf *
                                pip3 install https://github.com/sophgo/tpu-perf/releases/download/v1.1.5/tpu_perf-1.1.5-py3-none-manylinux2014_aarch64.whl --trusted-host mirrors.aliyun.com -i https://mirrors.aliyun.com/pypi/simple/
                                cd ../x86-agent-workspace/${X86_JOB_NAME}/${params.MODEL_ZOO_REPO}
                                python3 -m tpu_perf.run --mlir --list full_cases.txt
                          """
                        }
                    }
                }
                stage("Save-Report-To-Database") {
                    when {
                        expression {
                            return params.COMMIT_TO_DATASET
                        }
                    }
                    environment {
                        PGDB_CREDS = credentials('postgres-user')
                    }
                    steps {
                        dir("$WORKSPACE") {
                            sh """#!/bin/bash
                               set -e

                               cd ../x86-agent-workspace/${X86_JOB_NAME}/${params.MODEL_ZOO_REPO}

                               export PGPASSWORD=$PGDB_CREDS_PSW

                               psql -h 172.28.3.47 -p 8083 -U $PGDB_CREDS_USR -d tpu_mlir \\
                                    -c "\\COPY bm1684x_performance(name,shape,gops,\\"time(ms)\\",mac_utilization,cpu_usage,ddr_utilization) FROM './output/stats.csv' DELIMITER ',' CSV HEADER;"

                               psql -h 172.28.3.47 -p 8083 -U $PGDB_CREDS_USR -d tpu_mlir \\
                                    -c "UPDATE bm1684x_performance SET tpu_mlir_sha='$TPU_MLIR_SHA' WHERE tpu_mlir_sha IS NULL;"

                               psql -h 172.28.3.47 -p 8083 -U $PGDB_CREDS_USR -d tpu_mlir \\
                                    -c "UPDATE bm1684x_performance SET model_zoo_sha='$MODEL_ZOO_SHA' WHERE model_zoo_sha IS NULL;"

                               psql -h 172.28.3.47 -p 8083 -U $PGDB_CREDS_USR -d tpu_mlir \\
                                    -c "UPDATE bm1684x_performance SET date=CURRENT_TIMESTAMP WHERE date IS NULL;"

                               psql -h 172.28.3.47 -p 8083 -U $PGDB_CREDS_USR -d tpu_mlir -c \\
                                    "WITH v_tmp_table AS \\
                                    ( \\
                                        SELECT avg(\\"time(ms)\\") OVER (PARTITION BY name, shape) AS t_ag, id \\
                                        FROM bm1684x_performance \\
                                    ) \\
                                    UPDATE bm1684x_performance SET \\"time_avg(ms)\\" = v_tmp_table.t_ag \\
                                    FROM v_tmp_table \\
                                    WHERE bm1684x_performance.id = v_tmp_table.id;"
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