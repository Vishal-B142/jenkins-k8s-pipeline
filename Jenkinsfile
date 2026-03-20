pipeline {
    agent any
    tools {
        jdk 'jdk21'
        nodejs 'node20'
    }
    environment {
        AWS_REGION   = 'ap-south-1'
        ECR_REGISTRY = '<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com'
        ECR_REPO     = 'app'
        NAMESPACE    = 'app'
    }
    stages {

        /* =========================
           CHECKOUT
        ========================= */
        stage('Checkout') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh """
                        find . -mindepth 1 -delete || true
                        git clone --no-single-branch \\
                            https://\$GIT_USER:\$GIT_PASS@github.com/<YOUR_ORG>/<YOUR_REPO>.git .
                        git checkout dev
                    """
                }
            }
        }

        /* =========================
           INSTALL DEPENDENCIES
        ========================= */
        stage('Install Dependencies') {
            steps {
                sh 'npm ci --legacy-peer-deps'
            }
        }

        /* =========================
           DETECT AFFECTED APPS (Nx monorepo)
        ========================= */
        stage('Detect Affected Apps') {
            steps {
                script {
                    env.GIT_SHA = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    echo "Git SHA: ${env.GIT_SHA}"

                    withCredentials([usernamePassword(
                        credentialsId: 'github-credentials',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_PASS'
                    )]) {
                        sh """
                            git config credential.helper '!f() { echo username=\$GIT_USER; echo password=\$GIT_PASS; }; f'
                            git fetch origin dev:refs/remotes/origin/dev --depth=20
                        """
                    }

                    // List all deployable services here
                    def DEPLOYABLE_SERVICES = [
                        'auth-service',
                        'gateway-service',
                        'notification-service',
                        'payment-service',
                        // add your services
                    ]

                    def rawAffected = ""
                    try {
                        rawAffected = sh(
                            script: "npx nx show projects --affected --base=origin/dev --head=HEAD",
                            returnStdout: true
                        ).trim()
                        echo "Nx affected output:\n${rawAffected}"
                    } catch (Exception e) {
                        echo "WARNING: Nx affected detection failed: ${e.message}"
                        echo "Falling back to building ALL services."
                    }

                    if (!rawAffected?.trim()) {
                        echo "No affected projects detected — building ALL services."
                        env.AFFECTED_SERVICES = DEPLOYABLE_SERVICES.join('\n')
                    } else {
                        def filtered = rawAffected
                            .split('\n')
                            .collect { it.trim().replace('@app/', '') }
                            .findAll { it && DEPLOYABLE_SERVICES.contains(it) }
                        if (filtered.isEmpty()) {
                            echo "INFO: No deployable services affected. Skipping build."
                            env.AFFECTED_SERVICES = ''
                        } else {
                            env.AFFECTED_SERVICES = filtered.join('\n')
                            echo "Services to build and deploy:\n${env.AFFECTED_SERVICES}"
                        }
                    }
                }
            }
        }

        /* =========================
           BUILD & PUSH TO ECR
        ========================= */
        stage('Build & Push Images to ECR') {
            when {
                expression { env.AFFECTED_SERVICES?.trim() }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    script {
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} \\
                            | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        """
                        env.AFFECTED_SERVICES.split('\n').each { service ->
                            service = service.trim()
                            if (!service) return
                            def latestImage = "${ECR_REGISTRY}/${ECR_REPO}:${service}"
                            def shaImage    = "${ECR_REGISTRY}/${ECR_REPO}:${service}-${env.GIT_SHA}"
                            echo "Building: ${service}"
                            sh """
                                docker build \\
                                    --build-arg APP_NAME=${service} \\
                                    -f apps/${service}/Dockerfile \\
                                    -t ${latestImage} \\
                                    -t ${shaImage} .
                                docker push ${latestImage}
                                docker push ${shaImage}
                                echo "Pushed: ${shaImage}"
                            """
                        }
                    }
                }
            }
        }

        /* =========================
           REFRESH ECR SECRET IN CLUSTER
        ========================= */
        stage('Refresh ECR Secret') {
            when {
                expression { env.AFFECTED_SERVICES?.trim() }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    ),
                    file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG_FILE')
                ]) {
                    sh """
                        ECR_PASSWORD=\$(aws ecr get-login-password --region ${AWS_REGION})
                        kubectl --kubeconfig=\$KUBECONFIG_FILE delete secret ecr-secret -n ${NAMESPACE} --ignore-not-found
                        kubectl --kubeconfig=\$KUBECONFIG_FILE create secret docker-registry ecr-secret \\
                            --docker-server=${ECR_REGISTRY} \\
                            --docker-username=AWS \\
                            --docker-password=\$ECR_PASSWORD \\
                            -n ${NAMESPACE}
                        echo "ECR secret refreshed"
                    """
                }
            }
        }

        /* =========================
           APPLY K8S MANIFESTS
        ========================= */
        stage('Apply Manifests') {
            when {
                expression { env.AFFECTED_SERVICES?.trim() }
            }
            steps {
                withCredentials([
                    file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG_FILE')
                ]) {
                    sh """
                        kubectl --kubeconfig=\$KUBECONFIG_FILE apply -f k3s/deployments.yaml
                        # --validate=false: remote kubectl lacks Traefik CRD schemas
                        # cluster itself validates correctly
                        kubectl --kubeconfig=\$KUBECONFIG_FILE apply -f k3s/ingress.yaml --validate=false
                        kubectl --kubeconfig=\$KUBECONFIG_FILE apply -f k3s/hpa.yaml
                        echo "Manifests applied"
                    """
                }
            }
        }

        /* =========================
           BLUE/GREEN DEPLOY
        ========================= */
        stage('Deploy (Blue/Green)') {
            when {
                expression { env.AFFECTED_SERVICES?.trim() }
            }
            steps {
                withCredentials([
                    file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG_FILE')
                ]) {
                    script {
                        sh """
                            echo "=== Cluster Nodes ==="
                            kubectl --kubeconfig=\$KUBECONFIG_FILE get nodes
                        """
                        env.AFFECTED_SERVICES.split('\n').each { service ->
                            service = service.trim()
                            if (!service) return
                            def newImage = "${ECR_REGISTRY}/${ECR_REPO}:${service}-${env.GIT_SHA}"
                            echo "Blue/Green deploy: ${service} -> ${newImage}"
                            sh """
                                set -e

                                # Step 1: Determine active slot
                                ACTIVE=\$(kubectl --kubeconfig=\$KUBECONFIG_FILE \\
                                    get service ${service} -n ${NAMESPACE} \\
                                    -o jsonpath='{.metadata.annotations.active-slot}' 2>/dev/null || echo "blue")
                                if [ -z "\$ACTIVE" ]; then ACTIVE="blue"; fi

                                if [ "\$ACTIVE" = "blue" ]; then INACTIVE="green"; else INACTIVE="blue"; fi

                                echo "  Active:   \$ACTIVE"
                                echo "  Inactive: \$INACTIVE (deploying here)"

                                # Step 2: Set new image on inactive slot
                                kubectl --kubeconfig=\$KUBECONFIG_FILE \\
                                    set image deployment/${service}-\$INACTIVE \\
                                    ${service}=${newImage} -n ${NAMESPACE}

                                # Step 3: Scale up inactive slot
                                kubectl --kubeconfig=\$KUBECONFIG_FILE \\
                                    scale deployment/${service}-\$INACTIVE \\
                                    --replicas=1 -n ${NAMESPACE}

                                # Step 4: Wait for readiness (300s for large images)
                                kubectl --kubeconfig=\$KUBECONFIG_FILE \\
                                    rollout status deployment/${service}-\$INACTIVE \\
                                    -n ${NAMESPACE} --timeout=300s

                                echo "  ${service}-\$INACTIVE is healthy"

                                # Step 5: Switch Service selector to new slot
                                kubectl --kubeconfig=\$KUBECONFIG_FILE \\
                                    patch service ${service} -n ${NAMESPACE} \\
                                    -p "{\"spec\":{\"selector\":{\"app\":\"${service}\",\"slot\":\"\$INACTIVE\"}},\"metadata\":{\"annotations\":{\"active-slot\":\"\$INACTIVE\"}}}"

                                echo "  Service ${service} now points to \$INACTIVE"

                                # Step 6: Scale down old active slot
                                kubectl --kubeconfig=\$KUBECONFIG_FILE \\
                                    scale deployment/${service}-\$ACTIVE \\
                                    --replicas=0 -n ${NAMESPACE}

                                echo "  Scaled down ${service}-\$ACTIVE (now idle)"
                                echo "  ${service} deploy complete — active slot: \$INACTIVE"
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully"
        }
        failure {
            echo "Pipeline failed. Previous active slot is still running — zero downtime."
            echo "To rollback: patch the service selector back to the previous slot."
        }
        always {
            sh "docker logout ${ECR_REGISTRY} || true"
        }
    }
}
