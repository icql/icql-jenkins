pipeline {
    agent {
        node {
            label 'master'
        }
    }

    options {
        disableConcurrentBuilds()
        timeout(time: 10, unit: 'MINUTES')
    }

    triggers {
        GenericTrigger(
                genericVariables: [
                        [key: 'GIT_REF', value: '$.ref'],
                        [key: 'GIT_BEFORE', value: '$.before'],
                        [key: 'GIT_AFTER', value: '$.after']
                ],
                token: BRANCH_NAME,
                regexpFilterText: '$GIT_REF',
                regexpFilterExpression: 'refs/.'
        )
    }

    environment {
        GIT_REPONAME = "${BRANCH_NAME.split('-')[1]}-${BRANCH_NAME.split('-')[2]}"
        DOCKER_VOLUMES_WORKSPACE_PREFIX = '/data/icql-devops/jenkins/data/workspace'
        JENKINS_WORKSPACE_PREFIX = '/var/jenkins_home/workspace'
        DINGTALK_ROBOT = 'https://oapi.dingtalk.com/robot/send?access_token=8b7536c2584146d4e9d37a8e6b38352adc720990e82f4d09c###isd-400###'
    }

    stages {
        stage('ready') {
            when {
                expression {
                    env.GIT_REF != null && env.GIT_REF != '' && env.GIT_REF.contains('refs/')
                }
            }

            stages {
                stage('gitee pull') {
                    steps {
                        script {
                            //更新仓库文件
                            def gitDir = "${JENKINS_WORKSPACE_PREFIX}/00_ICQL/gitee-${GIT_REPONAME}";
                            if (fileExists(gitDir)) {
                                withCredentials([usernamePassword(
                                        credentialsId: "icql-account",
                                        usernameVariable: "GIT_USERNAME",
                                        passwordVariable: "GIT_PASSWORD")]) {
                                    sh "cd ${gitDir} && \
                                        git config --local credential.helper '!p() { sleep 5; echo username=${GIT_USERNAME}; echo password=${GIT_PASSWORD}; }; p' && \
                                        git config --local user.name 'icql' && \
                                        git config --local user.email 'chenqinglin@hnu.edu.cn'"
                                    sh "cd ${gitDir} && git pull origin master"
                                }
                            } else {
                                dir(gitDir) {
                                    git(
                                            url: "https://gitee.com/icql/${GIT_REPONAME}.git",
                                            credentialsId: "icql-account",
                                            branch: 'master'
                                    )
                                }
                            }
                            //获取最近同步的日志，复制hexo文件
                            sh "cd ${gitDir} \
                                && mkdir ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/tmp-git \
                                && (git log -5 --pretty=format:\"* [[%t]](https://gitee.com/icql/${GIT_REPONAME}/commit/%H) %s（%cn）\") > ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/tmp-git/${GIT_REPONAME}_LATEST_COMMITS \
                                && cp -R ${gitDir}/00_home/hexo ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/tmp-hexo"
                        }
                    }
                }

                stage('hexo generate') {
                    agent {
                        docker {
                            image 'icql/hexo:0.2.2'
                            args "-v ${DOCKER_VOLUMES_WORKSPACE_PREFIX}/00_ICQL/tmp-hexo:/hexo --entrypoint=''"
                        }
                    }
                    steps {
                        //hexo生成
                        sh 'sh /generate.sh'
                    }
                }

                stage('copy file') {
                    steps {
                        script {
                            //复制部署文件
                            sh "rm -rf ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/deploy-static/*"
                            sh "cp -R ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/gitee-${GIT_REPONAME}/* ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/deploy-static && \
                            cp -R ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/tmp-hexo/public ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/deploy-static/00_home/public"
                            sh "cp -R conf ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/deploy-static/conf"
                            //对外公开的文件处理敏感信息
                            withCredentials([usernamePassword(
                                    credentialsId: 'icql-secret-dictionary',
                                    usernameVariable: 'ICQL_SECRET_KEY',
                                    passwordVariable: 'ICQL_SECRET_VALUE')]) {
                                def secretWords = "${ICQL_SECRET_VALUE}".trim().tokenize('|')
                                for (secretWord in secretWords) {
                                    def secretWordKeyValue = secretWord.trim().tokenize(':')
                                    sh "cd ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/deploy-static/00_home/public && ((sed -i \"s/${secretWordKeyValue[1]}/******/g\" `grep ${secretWordKeyValue[1]} -rl ./`) || true)"
                                }
                            }
                        }
                    }
                }

                stage('deploy resources') {
                    agent {
                        docker {
                            image 'icql/helm-kubectl-docker:0.1.0'
                        }
                    }
                    steps {
                        withKubeConfig([credentialsId: 'icql-k8s-token', serverUrl: 'https://172.18.123.163:6443']) {
                            script {
                                def namespace = 'icql-static'
                                def app = 'nginx'
                                def sortMap = ['namespace'         : 1000,
                                               'serviceaccount'    : 1010,
                                               'clusterrolebinding': 1020,
                                               'apiservice'        : 1030,
                                               'configmap'         : 1040,
                                               'secret'            : 1050,
                                               'pv'                : 1060,
                                               'pvc'               : 1070,
                                               'deploy'            : 1111,
                                               'svc'               : 1112,
                                               'ingress'           : 1113]

                                def k8sResources = []
                                //还原配置文件中的secret信息
                                withCredentials([usernamePassword(
                                        credentialsId: 'icql-secret-dictionary',
                                        usernameVariable: 'ICQL_SECRET_KEY',
                                        passwordVariable: 'ICQL_SECRET_VALUE')]) {
                                    def secretWords = "${ICQL_SECRET_VALUE}".trim().tokenize('|')
                                    for (secretWord in secretWords) {
                                        def secretWordKeyValue = secretWord.trim().tokenize(':')
                                        sh "cd conf/${namespace} && ((sed -i \"s/${secretWordKeyValue[0]}/${secretWordKeyValue[1]}/g\" `grep ${secretWordKeyValue[0]} -rl ./`) || true)"
                                    }
                                }
                                //获取app对应的namespace资源文件路径
                                sh "ls conf/${namespace} | grep .yaml\$ > NS_${params.K8S_RESOURCES}"
                                def nsK8sResources = readFile("NS_${params.K8S_RESOURCES}").trim().tokenize('\n')
                                for (resource in nsK8sResources) {
                                    def resourceType = resource.tokenize('-')[0].replaceAll('.yaml', '')
                                    def resourceTypeSortValue = sortMap[resourceType]
                                    k8sResources.add("${resourceTypeSortValue}@conf/${namespace}/${resource}")
                                }
                                //获取app的资源文件路径
                                sh "ls conf/${namespace}/${app} | grep .yaml\$ > APP_${params.K8S_RESOURCES}"
                                def appK8sResources = readFile("APP_${params.K8S_RESOURCES}").trim().tokenize('\n')
                                for (resource in appK8sResources) {
                                    def resourceType = resource.tokenize('-')[0]
                                    def resourceTypeSortValue = sortMap[resourceType]
                                    k8sResources.add("${resourceTypeSortValue}@conf/${namespace}/${app}/${resource}")
                                }
                                //k8s资源根据依赖关系排序
                                def sortedK8sResources = k8sResources.sort()
                                println(sortedK8sResources)
                                //部署
                                for (resource in sortedK8sResources) {
                                    def resourceTypeSortValue = resource.tokenize('@')[0]
                                    def resourcePath = resource.tokenize('@')[1]
                                    sh "kubectl apply -f ${resourcePath}"
                                    if (resourceTypeSortValue == "1111") {
                                        sh "kubectl rollout status -f ${resourcePath}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            sh "rm -rf ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/tmp-hexo"
        }
        success {
            //执行成功
            sendMessage('成功')
        }
        failure {
            //执行失败
            sendMessage('失败')
        }
    }
}

def sendMessage(result) {
    //组装钉钉通知内容
    def latestCommits = ''
    if (fileExists("${JENKINS_WORKSPACE_PREFIX}/00_ICQL/tmp-git/${GIT_REPONAME}_LATEST_COMMITS")) {
        latestCommits = readFile("${JENKINS_WORKSPACE_PREFIX}/00_ICQL/tmp-git/${GIT_REPONAME}_LATEST_COMMITS").trim()
        sh "rm -rf ${JENKINS_WORKSPACE_PREFIX}/00_ICQL/tmp-git"
    }
    def message = "{\"msgtype\":\"markdown\",\"markdown\":{\"title\":\"DS 通知\",\"text\":\"" +
            "### [${JOB_NAME}/${BUILD_NUMBER}](${BUILD_URL}/console) ${result}\\n" +
            "#### 最近更新的内容：\\n" +
            "${latestCommits}\\n" +
            "\"}}"
    //发送钉钉通知
    withCredentials([usernamePassword(
            credentialsId: 'icql-secret-dictionary',
            usernameVariable: 'ICQL_SECRET_KEY',
            passwordVariable: 'ICQL_SECRET_VALUE')]) {
        def dingRobot
        def secretWords = "${ICQL_SECRET_VALUE}".trim().tokenize('|')
        for (secretWord in secretWords) {
            def secretWordKeyValue = secretWord.trim().tokenize(':')
            dingRobot = DINGTALK_ROBOT.replaceAll(secretWordKeyValue[0], secretWordKeyValue[1])
        }
        sh "curl ${dingRobot} -H 'Content-Type:application/json' -X POST --data '${message}'"
    }
}