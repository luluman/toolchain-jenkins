//===-*- Groovy -*-===

pipeline {
    agent any
    stage('Model-Zoo-Test') {
        build job: 'model-zoo-tpu-mlir-base', parameters: [
            string(name: 'TPU_MLIR_BRANCH', value: 'master'),
            string(name: 'MODEL_ZOO_REPO', value: 'model-zoo'),
            string(name: 'MODEL_ZOO_BRANCH', value: 'main'),
            booleanParam(name: 'COMMIT_TO_DATASET', value: true)
        ], wait: true, propagate: true
    }
}
