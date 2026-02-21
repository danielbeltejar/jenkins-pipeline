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
                image: 'mirror.gcr.io/alpine/helm'
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
                        echo "192.168.1.105 ${REGISTRY_URL}" | tee -a /etc/hosts
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
                                                    cat > /tmp/buildkitd.toml <<EOF
[registry."${REGISTRY_URL}"]
    ca=["/kaniko/.docker/certs/ca.crt"]
EOF
                                                    export BUILDKITD_FLAGS="$BUILDKITD_FLAGS --config /tmp/buildkitd.toml"
                                                fi

                        buildctl-daemonless.sh build \
                        --frontend dockerfile.v0 \
                        --local context=${dockerfileDir} \
                        --local dockerfile=${dockerfileDir} \
                        --opt filename=${dockerfileName} \
                        --import-cache type=registry,ref=${cacheRepo} \
                        --export-cache type=registry,ref=${cacheRepo},mode=max \
                        --output type=image,name=${imageWithVersion},name=${imageLatest},push=true
                        """

                        echo "BuildKit build completed for ${appName}:${IMAGE_VERSION_TAG}"
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
    }
}
