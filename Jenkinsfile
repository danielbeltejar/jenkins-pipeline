pipeline {
    options {
        timestamps()
    }

    agent {
        kubernetes {
            inheritFrom 'buildkit'
            defaultContainer 'buildkit'
            yaml """
            apiVersion: v1
            kind: Pod
            metadata:
            spec:
              hostAliases:
              - ip: "192.168.1.105"
                hostnames:
                - "harbor.server.local"
              containers:
              - name: buildkit
                image: 'moby/buildkit:v0.27.0-rootless'
                command:
                - sleep
                args:
                - infinity
                env:
                - name: BUILDKITD_FLAGS
                  value: --oci-worker-no-process-sandbox --oci-worker-snapshotter=native
                securityContext:
                    runAsUser: 1000
                    runAsGroup: 1000
                volumeMounts:
                - name: docker-config
                  mountPath: /kaniko/.docker
                - name: ca-certificate
                  mountPath: /kaniko/.docker/certs/
                - name: buildkit-state
                  mountPath: /home/user/.local/share/buildkit
              - name: helm
                image: 'mirror.gcr.io/alpine/k8s:1.32.3'
                command:
                - sleep
                args:
                - infinity
              - name: git
                image: 'mirror.gcr.io/alpine/git'
                command:
                - sleep
                args:
                - infinity
              - name: trivy
                image: 'aquasec/trivy:0.69.3'
                command:
                - sleep
                args:
                - infinity
                volumeMounts:
                - name: docker-config
                  mountPath: /kaniko/.docker
                - name: ca-certificate
                  mountPath: /kaniko/.docker/certs/
                - name: trivy-cache
                  mountPath: /root/.cache/trivy
              restartPolicy: Never
              volumes:
              - name: docker-config
                configMap:
                  name: docker-auth-config
              - name: ca-certificate
                hostPath:
                  path: /nfs/lab-jenkins/certs/
              - name: buildkit-state
                emptyDir: {}
              - name: trivy-cache
                emptyDir: {}
            """
        }
    }

    parameters {
        string(name: 'BUILD_ROOT', defaultValue: 'false', description: 'Build from repo root instead of subdir (true or false)')
    }

    environment {
        GIT_CREDENTIALS = credentials('GITHUB_AUTH_TOKEN')
        DISCORD_CREDENTIALS = credentials('DISCORD_CREDENTIALS')
        HARBOR_CREDENTIALS = credentials('HARBOR_CREDENTIALS')
        
        GIT_URL = "https://github.com/${params.GIT_URL.split('github.com/')[1]}"
        APP_NAME = "${params.APP_NAME}"
        BUILD_ROOT = "${params.BUILD_ROOT}"
        
        REGISTRY_URL = "harbor.server.local"
        HELM_RELEASE_NAME = "${JOB_NAME.replaceAll("[^a-zA-Z0-9]", "-").toLowerCase()}"
        HELM_CHART_DIR = "k8s/"
        IMAGE_REPO = "${GIT_URL.tokenize("/")[-1].replaceAll(".git", "").toLowerCase()}"
        IMAGE_VERSION_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                container('git') {
                    checkout([$class: 'GitSCM',
                              branches: [[name: 'main']],
                              userRemoteConfigs: [[url: "${GIT_URL}", credentialsId: 'GITHUB_AUTH_TOKEN']]
                    ])
                }
            }
        }
        stage('Configure Environment') {
            steps {
                container('buildkit') {
                    script {
                        sh '''
                        if grep -q "${REGISTRY_URL}" /etc/hosts; then
                            echo "Registry host alias present: ${REGISTRY_URL}"
                        else
                            echo "Warning: ${REGISTRY_URL} not present in /etc/hosts"
                        fi
                        '''
                    }
                }
            }
        }
        stage('Build Docker Image') {
            when {
                expression { env.APP_NAME != 'apigw' }
            }
            steps {
                container('buildkit') {
                    script {
                        def buildRoot = params.BUILD_ROOT == 'true'
                        def appDir = buildRoot ? '.' : APP_NAME
                        def appName = APP_NAME
                        def workspaceDir = env.WORKSPACE
                        def cacheRepo = "${REGISTRY_URL}/danielbeltejar/${IMAGE_REPO}/${appName}-cache"
                        def dockerfilePath = buildRoot ? "${workspaceDir}/Dockerfile" : "${workspaceDir}/${APP_NAME}/Dockerfile"
                        def dockerfileDir = dockerfilePath.substring(0, dockerfilePath.lastIndexOf('/'))
                        def dockerfileName = dockerfilePath.substring(dockerfilePath.lastIndexOf('/') + 1)
                        def imageWithVersion = "${REGISTRY_URL}/danielbeltejar/${IMAGE_REPO}/${appName}:${IMAGE_VERSION_TAG}"
                        def imageLatest = "${REGISTRY_URL}/danielbeltejar/${IMAGE_REPO}/${appName}:latest"
                        
                        sh """
                        cd ${appDir}
                        export DOCKER_CONFIG=/kaniko/.docker

                        if [ -f /kaniko/.docker/certs/ca.crt ]; then
                            if [ -f /etc/ssl/certs/ca-certificates.crt ]; then
                                cat /etc/ssl/certs/ca-certificates.crt /kaniko/.docker/certs/ca.crt > /tmp/ssl-ca-bundle.crt
                            elif [ -f /etc/ssl/cert.pem ]; then
                                cat /etc/ssl/cert.pem /kaniko/.docker/certs/ca.crt > /tmp/ssl-ca-bundle.crt
                            else
                                cp /kaniko/.docker/certs/ca.crt /tmp/ssl-ca-bundle.crt
                            fi
                            export SSL_CERT_FILE=/tmp/ssl-ca-bundle.crt

                            cat > /tmp/buildkitd.toml <<EOF
[registry."${REGISTRY_URL}"]
    ca=["/kaniko/.docker/certs/ca.crt"]
EOF
                            export BUILDKITD_FLAGS="\$BUILDKITD_FLAGS --config /tmp/buildkitd.toml"
                        fi

                        buildctl-daemonless.sh build \
                        --frontend dockerfile.v0 \
                        --local context=${dockerfileDir} \
                        --local dockerfile=${dockerfileDir} \
                        --opt filename=${dockerfileName} \
                        --import-cache type=registry,ref=${cacheRepo} \
                        --export-cache type=registry,ref=${cacheRepo},mode=max \
                        --output type=image,name=${imageWithVersion},push=true
                        """

                        echo "BuildKit build completed for ${appName}:${IMAGE_VERSION_TAG}"
                    }
                }
            }
        }
        stage('Security Scan') {
            steps {
                container('trivy') {
                    script {
                        def buildRoot = params.BUILD_ROOT == 'true'
                        def appDir = buildRoot ? '.' : APP_NAME
                        def appName = APP_NAME

                        // Configure custom CA certificates for Harbor registry
                        sh '''
                        if [ -f /kaniko/.docker/certs/ca.crt ]; then
                            if [ -f /etc/ssl/certs/ca-certificates.crt ]; then
                                cat /etc/ssl/certs/ca-certificates.crt /kaniko/.docker/certs/ca.crt > /tmp/combined-ca.crt
                            elif [ -f /etc/ssl/cert.pem ]; then
                                cat /etc/ssl/cert.pem /kaniko/.docker/certs/ca.crt > /tmp/combined-ca.crt
                            else
                                cp /kaniko/.docker/certs/ca.crt /tmp/combined-ca.crt
                            fi
                            export SSL_CERT_FILE=/tmp/combined-ca.crt
                        fi
                        '''

                        // IaC misconfiguration scan (Dockerfile + Helm chart)
                        echo "=== IaC Misconfiguration Scan ==="
                        sh """
                        if [ -f /tmp/combined-ca.crt ]; then export SSL_CERT_FILE=/tmp/combined-ca.crt; fi
                        trivy config --severity MEDIUM,HIGH,CRITICAL ${appDir} || true
                        """

                        echo "=== IaC Misconfiguration Gate (CRITICAL) ==="
                        sh """
                        if [ -f /tmp/combined-ca.crt ]; then export SSL_CERT_FILE=/tmp/combined-ca.crt; fi
                        trivy config --severity CRITICAL --exit-code 1 --skip-check-update ${appDir}
                        """

                        // Container image vulnerability scan (skip for apigw)
                        if (env.APP_NAME != 'apigw') {
                            def imageWithVersion = "${REGISTRY_URL}/danielbeltejar/${IMAGE_REPO}/${appName}:${IMAGE_VERSION_TAG}"

                            echo "=== Image Vulnerability Scan: ${imageWithVersion} ==="
                            sh """
                            if [ -f /tmp/combined-ca.crt ]; then export SSL_CERT_FILE=/tmp/combined-ca.crt; fi
                            export DOCKER_CONFIG=/kaniko/.docker
                            trivy image \\
                                --severity HIGH,CRITICAL \\
                                --ignore-unfixed \\
                                --scanners vuln \\
                                --no-progress \\
                                ${imageWithVersion} || true
                            """

                            echo "=== Image Vulnerability Gate (CRITICAL) ==="
                            sh """
                            if [ -f /tmp/combined-ca.crt ]; then export SSL_CERT_FILE=/tmp/combined-ca.crt; fi
                            export DOCKER_CONFIG=/kaniko/.docker
                            trivy image \\
                                --severity CRITICAL \\
                                --ignore-unfixed \\
                                --exit-code 1 \\
                                --scanners vuln \\
                                --no-progress \\
                                --skip-db-update \\
                                ${imageWithVersion}
                            """
                        } else {
                            echo "Image vulnerability scan skipped for APP_NAME=apigw"
                        }

                        echo "Security scan completed successfully."
                    }
                }
            }
        }
        stage('Package Helm Chart') {
            steps {
                container('helm') {
                    script {
                        echo "Starting Helm packaging for ${APP_NAME}:${IMAGE_VERSION_TAG}"
                        def buildRoot = params.BUILD_ROOT == 'true'
                        def appDir = buildRoot ? '.' : APP_NAME
                        sh """
                        cd ${appDir}
                        sed -i 's/^appVersion:.*/appVersion: ${IMAGE_VERSION_TAG}/' ${HELM_CHART_DIR}/Chart.yaml
                        helm package ${HELM_CHART_DIR} --version=${IMAGE_VERSION_TAG} --app-version=${IMAGE_VERSION_TAG}"""
                    }
                }
            }
        }
        stage('Upload Helm Package to GitHub') {
            steps {
                lock(resource: 'helm-charts') {
                    container('git') {
                        script {
                            echo "Starting Helm chart upload for ${APP_NAME}:${IMAGE_VERSION_TAG}"
                            def buildRoot = params.BUILD_ROOT == 'true'
                            def appDir = buildRoot ? '.' : APP_NAME
                            def appName = APP_NAME
                            sh """
                            cd ${appDir}
                            git config --global user.email "jenkins@ci.local"
                            git config --global user.name "Jenkins"
                            git clone https://${GIT_CREDENTIALS}@github.com/danielbeltejar/helm-charts.git helm-charts
                            mkdir -p helm-charts/charts/app/${IMAGE_REPO}/${appName}/
                            rm -rf helm-charts/charts/app/${IMAGE_REPO}/${appName}/*
                            cp -rf ${HELM_CHART_DIR}* helm-charts/charts/app/${IMAGE_REPO}/${appName}/
                            cd helm-charts
                            git add charts/app/${IMAGE_REPO}/${appName}
                            git commit -m "Add Helm package for ${IMAGE_REPO}-${appName} version ${IMAGE_VERSION_TAG}"
                            git push origin develop
                            """
                        }
                    }
                }
            }
        }
        stage('Deploy with Helm') {
            steps {
                container('helm') {
                    script {
                        echo "Starting Helm deploy for ${APP_NAME}:${IMAGE_VERSION_TAG}"
                        def buildRoot = params.BUILD_ROOT == 'true'
                        def appDir = buildRoot ? '.' : APP_NAME
                        def appName = APP_NAME
                        sh """
                        cd ${appDir}
                        helm upgrade --install ${HELM_RELEASE_NAME} ${appName}-${IMAGE_VERSION_TAG}.tgz --set registry=${REGISTRY_URL}/danielbeltejar/${IMAGE_REPO}
                        """
                    }
                }
            }
        }
        stage('Verify Deployment') {
            steps {
                container('helm') {
                    script {
                        def rolloutTimeout = '300s'
                        def workloadTypes = ['Deployment', 'StatefulSet', 'DaemonSet']
                        def namespace = sh(script: "cat /var/run/secrets/kubernetes.io/serviceaccount/namespace 2>/dev/null || echo 'default'", returnStdout: true).trim()

                        echo "Verifying rollout for release '${HELM_RELEASE_NAME}' in namespace '${namespace}'"

                        def manifest = sh(script: "helm get manifest ${HELM_RELEASE_NAME}", returnStdout: true).trim()

                        def workloads = []
                        def currentKind = ''
                        manifest.eachLine { line ->
                            if (line =~ /^kind:\s+(.+)/) {
                                currentKind = (line =~ /^kind:\s+(.+)/)[0][1].trim()
                            }
                            if (line =~ /^\s+name:\s+(.+)/ && workloadTypes.contains(currentKind)) {
                                def name = (line =~ /^\s+name:\s+(.+)/)[0][1].trim()
                                workloads << [kind: currentKind, name: name]
                                currentKind = ''
                            }
                        }
                        workloads = workloads.unique()

                        if (workloads.isEmpty()) {
                            echo "WARNING: No Deployment, StatefulSet, or DaemonSet found in release '${HELM_RELEASE_NAME}'. Skipping rollout verification."
                            return
                        }

                        echo "Found ${workloads.size()} workload(s) to verify: ${workloads.collect { it.kind + '/' + it.name }.join(', ')}"

                        try {
                            workloads.each { wl ->
                                echo "Waiting for rollout: ${wl.kind}/${wl.name} (timeout: ${rolloutTimeout})"
                                sh "kubectl rollout status ${wl.kind.toLowerCase()}/${wl.name} -n ${namespace} --timeout=${rolloutTimeout}"
                            }

                            echo "All workloads rolled out successfully."
                            sh "kubectl get pods -n ${namespace} -l app.kubernetes.io/instance=${HELM_RELEASE_NAME} -o wide || true"

                        } catch (Exception e) {
                            echo "ROLLOUT VERIFICATION FAILED: ${e.getMessage()}"

                            echo "=== Pod Status ==="
                            sh "kubectl get pods -n ${namespace} -l app.kubernetes.io/instance=${HELM_RELEASE_NAME} -o wide || true"

                            echo "=== Pod Events ==="
                            workloads.each { wl ->
                                sh "kubectl describe ${wl.kind.toLowerCase()}/${wl.name} -n ${namespace} | tail -30 || true"
                            }
                            sh "kubectl get pods -n ${namespace} -l app.kubernetes.io/instance=${HELM_RELEASE_NAME} -o jsonpath='{range .items[*]}{.metadata.name}{\"\\n\"}{end}' | while read pod; do echo \"--- Events for pod: \$pod ---\"; kubectl describe pod \$pod -n ${namespace} | grep -A 20 'Events:' || true; done"

                            echo "=== Recent Namespace Events ==="
                            sh "kubectl get events -n ${namespace} --sort-by=.lastTimestamp | tail -20 || true"

                            echo "Rolling back release '${HELM_RELEASE_NAME}' to previous revision..."
                            sh "helm rollback ${HELM_RELEASE_NAME} 0 --wait --timeout ${rolloutTimeout}"
                            echo "Rollback completed."

                            error("Deployment verification failed for release '${HELM_RELEASE_NAME}'. Rollback executed. Check logs above for diagnostics.")
                        }
                    }
                }
            }
        }
    }
}
