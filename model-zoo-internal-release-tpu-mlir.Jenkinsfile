//===-*- Groovy -*-===

pipeline {
    agent any
    stages {
        stage('Model-Zoo-Internal-Test') {
            steps {
                build job: 'model-zoo-tpu-mlir-base', parameters: [
                    string(name: 'TPU_MLIR_BRANCH', value: 'release'),
                    string(name: 'MODEL_ZOO_REPO', value: 'model-zoo-internal'),
                    string(name: 'MODEL_ZOO_BRANCH', value: 'release'),
                    booleanParam(name: 'COMMIT_TO_DATASET', value: false)
                ], wait: true, propagate: true
            }
        }
    }
}
