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
        DINGTALK_ROBOT = 'https://oapi.dingtalk.com/robot/send?access_token=8b7536c2584146d4e9d37a8e6b38352adc720990e82f4d09c###isd-400###'
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
                                    def secretWords = "${ICQL_SECRET_VALUE}".trim().tokenize('|')
                                    for (secretWord in secretWords) {
                                        def secretWordKeyValue = secretWord.trim().tokenize(':')
                                        sh "sed -i \"s/${secretWordKeyValue[0]}/${secretWordKeyValue[1]}/g\" `grep \"${secretWordKeyValue[0]}\" -rl ./conf/${namespace}` || true"
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
    //组装钉钉通知内容
    def message = "{\"msgtype\":\"markdown\",\"markdown\":{\"title\":\"DK 通知\",\"text\":\"" +
            "### [${JOB_NAME}/${BUILD_NUMBER}](${BUILD_URL}/console) ${result}\\n" +
            "#### 部署的资源：\\n" +
            "* ${params.K8S_RESOURCES}\\n" +
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