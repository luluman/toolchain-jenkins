//===-*- Groovy -*-===

pipeline {
    agent any
    stages {
        stage('Pull Gerrit') {
            steps {

                script {
                    def projects = ['tpu-mlir',
                                    'nnmodels',
                                    'model-zoo-internal',
                                    'model-zoo'
                    ]

                    for (int i = 0; i < projects.size(); ++i) {
                        withCredentials([gitUsernamePassword(credentialsId: '5ca28ee4-187f-4ce5-b19d-f8c805ffb7a4', gitToolName: 'git-tool')]) {
                            sh """#!/bin/bash
                                set -e
                                cd /git-repository
                                export GIT_SSL_NO_VERIFY=1
                                if [ -d ${projects[i]} ];then
                                    cd ${projects[i]}
                                    echo $PWD
                                    git-lfs uninstall
                                    git clean -ffdx
                                    git reset --hard origin/\$(git branch --show-current)
                                    git remote update
                                    git-lfs install
                                    set +e
                                    git lfs fetch --all
                                    git lfs pull --include="*" --exclude=""
                                    ret=\$?
                                    set -e
                                    if [ \$ret -eq 2 ];then
                                        echo "failed to fetch some objects!"
                                        exit 0
                                     fi
                                     exit \$ret
                                elif [ -d ${projects[i]}.git ];then
                                    cd ${projects[i]}.git
                                    git remote update && git lfs fetch --all
                                else
                                    git clone --mirror https://gerrit-ai.sophgo.vip:8443/a/${projects[i]}
                                    cd ${projects[i]}.git
                                    git lfs fetch --all
                                fi
                            """
                        }
                    }
                }
            }
        }
    }
}
