pipeline {
    agent {
        kubernetes {
            inheritFrom 'kaniko'
            defaultContainer 'kaniko'
            yaml """
            apiVersion: v1
            kind: Pod
            metadata:
            spec:
              containers:
              - name: kaniko
                image: 'gcr.io/kaniko-project/executor:debug'
                command:
                - sleep
                args:
                - infinity
                volumeMounts:
                - name: docker-config
                  mountPath: /kaniko/.docker
                - name: ca-certificate
                  mountPath: /kaniko/.docker/certs/
              - name: helm
                image: 'alpine/helm'
                command:
                - sleep
                args:
                - infinity
              - name: git
                image: 'cgr.dev/chainguard/git:latest'
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
            """
        }
    }

    environment {
        GIT_CREDENTIALS = credentials('GITHUB_AUTH_TOKEN')
        DISCORD_CREDENTIALS = credentials('DISCORD_CREDENTIALS')
        HARBOR_CREDENTIALS = credentials('HARBOR_CREDENTIALS')
        
        GIT_URL = "https://${GIT_CREDENTIALS}@github.com/${params.GIT_URL.split('github.com/')[1]}"
        APP_NAME = "${params.APP_NAME}"
        
        APP_DIRECTORY = "${APP_NAME}"
        REGISTRY_URL = "harbor.server.local"
        HELM_RELEASE_NAME = "${JOB_NAME.replaceAll("[^a-zA-Z0-9]", "-").toLowerCase()}"
        HELM_CHART_DIR = "k8s/"
        IMAGE_REPO = "${GIT_URL.tokenize("/")[-1].replaceAll(".git", "").toLowerCase()}"
        IMAGE_VERSION_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                container('kaniko') {
                    checkout([$class: 'GitSCM',
                              branches: [[name: 'main']],
                              userRemoteConfigs: [[url: "${GIT_URL}"]]
                    ])
                }
            }
        }
        stage('Configure Environment') {
            steps {
                container('kaniko') {
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
                container('kaniko') {
                    script {
                        def dockerfileContent = readFile("${APP_NAME}/Dockerfile")
                        def fromCount = dockerfileContent.split('\n').findAll { it.trim().startsWith('FROM') }.size()
                        def ignorePathOption = fromCount > 1 ? '--ignore-path /' : ''
                        
                        sh """
                        cd ${APP_NAME}
                        /kaniko/executor \
                        --context=`pwd` \
                        --dockerfile=`pwd`/Dockerfile \
                        --destination=${REGISTRY_URL}/danielbeltejar/${IMAGE_REPO}/${APP_NAME}:${IMAGE_VERSION_TAG} \
                        --destination=${REGISTRY_URL}/danielbeltejar/${IMAGE_REPO}/${APP_NAME}:latest \
                        --cache=false \
                        --cache-dir=/cache \
                        --snapshot-mode=redo \
                        --registry-certificate "${REGISTRY_URL}=/kaniko/.docker/certs/ca.crt" \
                        ${ignorePathOption}
                        """
                    }
                }
            }
        }
        stage('Package Helm Chart') {
            steps {
                container('helm') {
                    script {
                        sh """
                        cd ${APP_NAME}
                        helm package ${HELM_CHART_DIR} --version=${IMAGE_VERSION_TAG} --app-version=${IMAGE_VERSION_TAG}"""
                    }
                }
            }
        }
        stage('Upload Helm Package to GitHub') {
            steps {
                container('git') {
                    script {
                        sh """
                        cd ${APP_NAME}
                        git config --global user.email "jenkins@ci.local"
                        git config --global user.name "Jenkins"
                        git clone https://${GIT_CREDENTIALS}@github.com/danielbeltejar/helm-charts.git helm-charts-repo
                        cp ${APP_NAME}-${IMAGE_VERSION_TAG}.tgz helm-repo/charts/${APP_NAME}.tgz
                        cd helm-repo
                        git add charts/${APP_NAME}.tgz
                        git commit -m "Add Helm package for ${APP_NAME} version ${IMAGE_VERSION_TAG}"
                        git push origin develop
                        """
                    }
                }
            }
        }
        stage('Deploy with Helm') {
            steps {
                container('helm') {
                    script {
                        sh """
                        cd ${APP_NAME}
                        helm upgrade --install ${HELM_RELEASE_NAME} ${APP_NAME}-${IMAGE_VERSION_TAG}.tgz --set registry=${REGISTRY_URL}/danielbeltejar/${IMAGE_REPO}
                        """
                    }
                }
            }
        }
    }
}
