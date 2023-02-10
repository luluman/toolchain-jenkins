//===-*- Groovy -*-===

pipeline {
    agent any
    stage('Model-Zoo-Internal-Test') {
        build job: 'model-zoo-tpu-mlir-base', parameters: [
            string(name: 'TPU_MLIR_BRANCH', value: 'master'),
            string(name: 'MODEL_ZOO_REPO', value: 'model-zoo-internal'),
            string(name: 'MODEL_ZOO_BRANCH', value: 'master'),
            booleanParam(name: 'COMMIT_TO_DATASET', value: false)
        ], wait: true, propagate: true
    }
}
