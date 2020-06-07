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

    parameters {
        choice(name: 'K8S_RESOURCES',
                choices: [
                        '',
                        'icql-test_nginx',

                        'icql-svc_mysql',

                        'icql-devops_frp',
                        'icql-devops_jenkins',

                        'kube-system_dashboard'//,
                        //'kube-system_metrics',
                        //'kube-system_ingress'
                ],
                description: '请选择部署的k8s资源')
    }

    environment {
        DOCKER_VOLUMES_WORKSPACE_PREFIX = '/data/icql-devops/jenkins/data/workspace'
        JENKINS_WORKSPACE_PREFIX = '/var/jenkins_home/workspace'
        DINGTALK_ROBOT_URL = 'https://oapi.dingtalk.com/robot/send?access_token=8b7536c2584146d4e9d37a8e6b38352adc720990e82f4d09c###isd-400###'
        WECHAT_ROBOT_ACCESS_TOKEN_URL = 'https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid=###isd-500###&corpsecret=###isd-510###'
        WECHAT_ROBOT_URL = 'https://qyapi.weixin.qq.com/cgi-bin/message/send?access_token='
    }

    stages {
        stage('ready') {
            when {
                expression {
                    params.K8S_RESOURCES != ''
                }
            }

            stages {
                stage('deploy resources') {
                    agent {
                        docker {
                            image 'icql/helm-kubectl-docker:0.1.0'
                        }
                    }
                    steps {
                        sh 'echo 开始'
                        withKubeConfig([credentialsId: 'icql-k8s-token', serverUrl: 'https://172.18.123.163:6443']) {
                            script {
                                def namespace = params.K8S_RESOURCES.split('_')[0]
                                def app = params.K8S_RESOURCES.split('_')[1]
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
                                    def secretWordsMap = evaluate(ICQL_SECRET_VALUE.replaceAll("\\{", "[").replaceAll("\\}", "]"))
                                    secretWordsMap.each { key, value ->
                                        sh "sed -i \"s/${key}/${value}/g\" `grep \"${key}\" -rl ./conf/${namespace}` || true"
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
                                    if (resourceTypeSortValue == "1111") {
                                        //判断deploy资源是否需要replace
                                        def needReplace = false
                                        if (fileExists('tmp-RESOURCE_STATUSES')) {
                                            def resourceStatuses = readFile('tmp-RESOURCE_STATUSES').trim().tokenize('\n')
                                            for (resourceStatus in resourceStatuses) {
                                                if (resourceStatus.endsWith('configured') || resourceStatus.endsWith('created')) {
                                                    needReplace = true
                                                    break
                                                }
                                            }
                                        }
                                        if (needReplace) {
                                            sh "kubectl replace -f ${resourcePath} >> tmp-RESOURCE_STATUSES"
                                        } else {
                                            sh "kubectl apply -f ${resourcePath} >> tmp-RESOURCE_STATUSES"
                                        }
                                        sh "kubectl rollout status -f ${resourcePath}"
                                    } else {
                                        sh "(kubectl apply -f ${resourcePath}) >> tmp-RESOURCE_STATUSES"
                                    }
                                }
                                sh 'cat tmp-RESOURCE_STATUSES'
                                sh 'rm -rf tmp-*'
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            //删除缓存文件
            sh 'rm -rf tmp*'
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
    withCredentials([usernamePassword(
            credentialsId: 'icql-secret-dictionary',
            usernameVariable: 'ICQL_SECRET_KEY',
            passwordVariable: 'ICQL_SECRET_VALUE')]) {
        def secretWordsMap = evaluate(ICQL_SECRET_VALUE.replaceAll("\\{", "[").replaceAll("\\}", "]"))

        //组装钉钉通知内容
        def dingMessage = "{\"msgtype\":\"markdown\",\"markdown\":{\"title\":\"DK 通知\",\"text\":\"" +
                "### [${JOB_NAME}/${BUILD_NUMBER}](${BUILD_URL}/console) ${result}\\n" +
                "#### 部署的资源：\\n" +
                "* ${params.K8S_RESOURCES}\\n" +
                "\"}}"
        //发送钉钉通知
        def dingRobot = DINGTALK_ROBOT_URL.replaceAll("###isd-400###", secretWordsMap["###isd-400###"])
        sh "curl ${dingRobot} -H 'Content-Type:application/json' -X POST --data '${dingMessage}'"

        //组装微信通知内容
        def wechatMessage = "{\n" +
                "    \"toparty\": \"1\",\n" +
                "    \"agentid\": ${secretWordsMap["###isd-511###"]},\n" +
                "    \"msgtype\": \"news\",\n" +
                "    \"news\": {\n" +
                "        \"articles\": [\n" +
                "            {\n" +
                "                \"title\": \"${JOB_NAME.tokenize('/')[0]}${result}\",\n" +
                "                \"description\": \"${JOB_NAME}/${BUILD_NUMBER}\n部署的资源：${params.K8S_RESOURCES}\",\n" +
                "                \"url\": \"${BUILD_URL}\",\n" +
                "                \"picurl\": \"https://file.icql.work/30_picture/1002_jenkins.jpg\"\n" +
                "            }\n" +
                "        ]\n" +
                "    },\n" +
                "    \"safe\": 0,\n" +
                "    \"enable_id_trans\": 0,\n" +
                "    \"enable_duplicate_check\": 0,\n" +
                "    \"duplicate_check_interval\": 1800\n" +
                "}"
        //获取微信应用access_token
        def wechatRobotAccessTokenUrl = WECHAT_ROBOT_ACCESS_TOKEN_URL.replaceAll("###isd-500###", secretWordsMap["###isd-500###"]).replaceAll("###isd-510###", secretWordsMap["###isd-510###"])
        sh "curl -s -- \"${wechatRobotAccessTokenUrl}\" > tmp-WECHAT_ROBOT_ACCESS_TOKEN"
        def wechatRobotAccessToken = evaluate(readFile('tmp-WECHAT_ROBOT_ACCESS_TOKEN').trim().replaceAll("\\{", "[").replaceAll("\\}", "]"))["access_token"]
        sh 'rm -rf tmp-*'
        //发送微信消息通知
        if (wechatRobotAccessToken != null && wechatRobotAccessToken != '') {
            sh "curl ${WECHAT_ROBOT_URL}${wechatRobotAccessToken} -H 'Content-Type:application/json' -X POST --data '${wechatMessage}'"
        }
    }
}