pipeline {
    agent any // This ensures a node/executor is allocated
    stages {
        stage('Build') {
            steps {
                script {
                    try {
                        // All 'sh' steps must happen inside an agent/node context. This will always fail without a makefile
                        sh 'make build'
                    } catch (err) {
                        // The 'node' context is still active here, so this 'sh' works
                        def lokiUrl = "http://loki.observability.svc.cluster.local:3100/loki/api/v1/push"
                        def timestamp = System.currentTimeMillis() * 1000000
                        
                        sh """
                        curl -X POST -H "Content-Type: application/json" \
                        -d '{"streams": [{"stream": {"job": "jenkins-pipeline"}, "values": [["${timestamp}", "Build Failed: ${err}"]]}]}' \
                        ${lokiUrl}
                        """
                        error("Build failed: ${err}")
                    }
                }
            }
        }
    }
}
