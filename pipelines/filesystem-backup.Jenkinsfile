pipeline {
    agent none

    parameters {
        choice(
            name: 'Environment',
            choices: ['env_not_selected', 'env_dev', 'env_test', 'env_stage'],
            description: 'Node Label'
        )
        string(name: 'REQUIRED_GB', defaultValue: '20', description: 'Required free space in GB')
    }

    stages {

        // Stage 1: Validate node availability
        stage('Stage 1: Validate node availability') {
            steps {
                script {
                    try {
                        timeout(time: 20, unit: 'SECONDS') {
                            node("${params.Environment}") {
                                echo "Node with label ${params.Environment} allocated"
                            }
                        }
                    } catch (err) {
                        error "No available node for label ${params.Environment} within 20 seconds"
                    }
                }
            }
        }

        // Stage 2: Validate disk space on ALL nodes
        stage('Stage 2: Validate disk space on ALL nodes') {
            steps {
                script {

                    def nodes = []

                    Jenkins.instance.nodes.each { n ->
                        if (n.getLabelString().contains(params.Environment)) {
                            nodes << n.getNodeName()
                        }
                    }

                    echo "Nodes found: ${nodes}"

                    def failedNodes = []

                    // VALIDATION
                    for (n in nodes) {
                        def nodeName = n   // FIX

                        node(nodeName) {
                            def status = sh(
                                script: """
                                    if [ -z "\${APP_HOME:-}" ]; then
                                        exit 2
                                    fi

                                    AVAILABLE_GB=\$(df -BG "\$APP_HOME" | awk 'NR==2 {gsub("G","",\$4); print \$4}')

                                    echo "Node ${nodeName} has \${AVAILABLE_GB}GB"

                                    if [ "\$AVAILABLE_GB" -lt "${params.REQUIRED_GB}" ]; then
                                        exit 1
                                    fi

                                    exit 0
                                """,
                                returnStatus: true
                            )

                            if (status != 0) {
                                failedNodes << nodeName
                            }
                        }
                    }

                    // Abort if any node fails
                    if (failedNodes.size() > 0) {
                        error "Backup ABORTED. Nodes without required space: ${failedNodes}"
                    }

                    echo "All nodes have enough space. Proceeding to backup..."
                }
            }
        }
   
        // Stage 3: Backup in parallel all nodes
        stage('Stage 3: Backup in parallel all nodes') {
            steps {
                script {
                    def nodes = []

                    Jenkins.instance.nodes.each { n ->
                        if (n.getLabelString().contains(params.Environment)) {
                            nodes << n.getNodeName()
                        }
                    }   

                    def branches = [:]

                    for (n in nodes) {
                        def nodeName = n

                        branches[nodeName] = {
                            node(nodeName) {

                                timeout(time: 3, unit: 'HOURS') {

                                    sh """
                                        echo "Running backup on ${nodeName}"

                                        trap "echo 'Stopping backup'; kill 0; exit 1" TERM INT

                                        if [ -z "\${APP_HOME:-}" ]; then
                                            echo "ERROR: APP_HOME not defined"
                                            exit 1
                                        fi

                                        cd "\$APP_HOME"

                                        DATE=\$(date +%Y%m%d)
                                        FILE="Backup_${nodeName}_${BUILD_NUMBER}_\$DATE.tar.gz"

                                        cp .profile profile_${BUILD_NUMBER}_\$DATE

                                        tar --ignore-failed-read \
                                            --exclude='data/server/archive/*' \
                                            --exclude='data/server/log/*' \
                                            --exclude='admin/releases/*' \
                                            --exclude='data/server/log.old/*' \
                                            --exclude='sv/*.tar.gz' \
                                            --exclude='data/server/config/tomcat/logs/*' \
                                            --exclude='data/server/invoices/images/*' \
                                            --exclude='data/client/*' \
                                            --exclude="*.sok" \
                                            -zcf "\$FILE" \
                                            <DIR_1> <DIR_2> <DIR_3> \
                                            profile_${BUILD_NUMBER}_\$DATE

                                        echo "Backup finished on ${nodeName}"

                                        ls -lh "\$APP_HOME/\$FILE" > "\$WORKSPACE/artifact_${nodeName}.txt"
                                    """
                                }
                            }
                        }
                    }

                    // CORRECT PLACE
                    parallel branches
                }
            }
        }

    }

    post {
        success {
            echo 'Backup stage completed'
            script {
                def nodes = []
                Jenkins.instance.nodes.each { n ->
                    if (n.getLabelString().contains(params.Environment)) {
                        nodes << n.getNodeName()
                    }
                }
                // Archive artifacts from EACH node
                for (n in nodes) {
                    node(n) {
                        archiveArtifacts artifacts: '*.txt', fingerprint: true, allowEmptyArchive: true
                    }
                }
            }
        }

        aborted {
            echo 'Node not available or timeout reached'
        }

        failure {
            echo 'Validation failed'
        }      
    }
}

    
