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
        DINGTALK_ROBOT_URL = 'https://oapi.dingtalk.com/robot/send'
        WECHAT_ROBOT_ACCESS_TOKEN_URL = 'https://qyapi.weixin.qq.com/cgi-bin/gettoken'
        WECHAT_ROBOT_URL = 'https://qyapi.weixin.qq.com/cgi-bin/message/send'
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
                        //设置git全局凭据
                        withCredentials([usernamePassword(
                                credentialsId: 'icql-account',
                                usernameVariable: 'GIT_USERNAME',
                                passwordVariable: 'GIT_PASSWORD')]) {
                            sh "git config --global credential.helper '!p() { sleep 5; echo username=${GIT_USERNAME}; echo password=${GIT_PASSWORD}; }; p'"
                        }
                        //拉取需要同步的仓库
                        sh "git clone --bare https://gitee.com/icql/${GIT_REPONAME}.git tmp-gitee-${GIT_REPONAME}"
                    }
                }
                stage('commit log') {
                    steps {
                        script {
                            def empty8 = '00000000'
                            def empty40 = '0000000000000000000000000000000000000000'
                            if (GIT_REF.contains('refs/heads/')) {
                                def GIT_BRANCH = GIT_REF.tokenize('/')[2]
                                if (GIT_BEFORE == empty40 || GIT_BEFORE == empty8) {
                                    //新增分支
                                    sh "echo \"* [[${GIT_AFTER.take(6)}]](https://gitee.com/icql/${GIT_REPONAME}/commit/${GIT_AFTER}) 新增分支${GIT_BRANCH}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS_MARKDOWN"
                                    sh "echo \"• [${GIT_AFTER.take(6)}] 新增分支${GIT_BRANCH}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS_TEXT"
                                } else if (GIT_AFTER == empty40 || GIT_AFTER == empty8) {
                                    //删除分支
                                    sh "echo \"* [[${GIT_BEFORE.take(6)}]](https://gitee.com/icql/${GIT_REPONAME}/commit/${GIT_BEFORE}) 删除分支${GIT_BRANCH}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS_MARKDOWN"
                                    sh "echo \"• [${GIT_BEFORE.take(6)}] 删除分支${GIT_BRANCH}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS_TEXT"
                                } else {
                                    //普通提交
                                    sh "cd tmp-gitee-${GIT_REPONAME} && \
                                    (git log ${GIT_BRANCH} -5 --pretty=format:\"* [[%t]](https://gitee.com/icql/${GIT_REPONAME}/commit/%H) %s（%cn）\") > ../${GIT_REPONAME}_LATEST_COMMITS_MARKDOWN && \
                                    (git log ${GIT_BRANCH} -5 --pretty=format:\"• [%t] %s（%cn）\") > ../${GIT_REPONAME}_LATEST_COMMITS_TEXT"
                                }
                            } else if (GIT_REF.contains('refs/tags/')) {
                                def GIT_TAG = GIT_REF.tokenize('/')[2]
                                if (GIT_BEFORE == empty40 || GIT_BEFORE == empty8) {
                                    //新增标签
                                    sh "echo \"* [[${GIT_AFTER.take(6)}]](https://gitee.com/icql/${GIT_REPONAME}/commit/${GIT_AFTER}) 新增标签${GIT_TAG}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS_MARKDOWN"
                                    sh "echo \"• [${GIT_AFTER.take(6)}] 新增标签${GIT_TAG}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS_TEXT"
                                } else if (GIT_AFTER == empty40 || GIT_AFTER == empty8) {
                                    //删除标签
                                    sh "echo \"* [[${GIT_BEFORE.take(6)}]](https://gitee.com/icql/${GIT_REPONAME}/commit/${GIT_BEFORE}) 删除标签${GIT_TAG}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS_MARKDOWN"
                                    sh "echo \"• [${GIT_BEFORE.take(6)}] 删除标签${GIT_TAG}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS_TEXT"
                                }
                            }
                        }
                    }
                }

                stage('github push') {
                    steps {
                        //同步github仓库
                        sh "cd tmp-gitee-${GIT_REPONAME} && \
                            git push --mirror https://github.com/icql/${GIT_REPONAME}.git"
                        //清除git全局凭据
                        sh 'git config --global --unset credential.helper'
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
        def latestCommitsMd = ''
        if (result == '成功' && fileExists("${GIT_REPONAME}_LATEST_COMMITS_MARKDOWN")) {
            latestCommitsMd = readFile("${GIT_REPONAME}_LATEST_COMMITS_MARKDOWN").trim().replaceAll("\n", "\\n")
        }
        sh "rm -rf ${GIT_REPONAME}_LATEST_COMMITS_MARKDOWN"
        def dingMessage = "{\"msgtype\":\"markdown\",\"markdown\":{\"title\":\"DS 通知\",\"text\":\"" +
                "### [${JOB_NAME}/${BUILD_NUMBER}](${BUILD_URL}/console) ${result}\\n" +
                "#### 最近同步的内容：\\n" +
                "${latestCommitsMd}\\n" +
                "\"}}"
        //发送钉钉通知
        def dingRobotUrl = "${DINGTALK_ROBOT_URL}?access_token=8b7536c2584146d4e9d37a8e6b38352adc720990e82f4d09c${secretWordsMap["###isd-400###"]}"
        sh "curl ${dingRobotUrl} -H 'Content-Type:application/json' -X POST --data '${dingMessage}'"


        //组装微信通知内容
        def latestCommitsText = ''
        if (result == '成功' && fileExists("${GIT_REPONAME}_LATEST_COMMITS_TEXT")) {
            latestCommitsText = readFile("${GIT_REPONAME}_LATEST_COMMITS_TEXT").trim().replaceAll("\n", "\\n")
        }
        sh "rm -rf ${GIT_REPONAME}_LATEST_COMMITS_TEXT"
        def wechatMessage = "{\"toparty\": \"1\",\"agentid\": ${secretWordsMap["###isd-511###"]},\"msgtype\": \"news\",\"news\": {\"articles\": [{\"title\": \"" +
                "${JOB_NAME.tokenize('/')[0]}${result}" +
                "\",\"description\": \"" +
                "${JOB_NAME}/${BUILD_NUMBER}\\n最近同步的内容：\\n${latestCommitsText}" +
                "\",\"url\": \"" +
                "${BUILD_URL}" +
                "\",\"picurl\": \"" +
                "https://file.icql.work/30_picture/1002_jenkins.jpg" +
                "\"}]},\"safe\": 0,\"enable_id_trans\": 0,\"enable_duplicate_check\": 0,\"duplicate_check_interval\": 1800}"
        //获取微信应用access_token
        def wechatRobotAccessTokenUrl = "${WECHAT_ROBOT_ACCESS_TOKEN_URL}?corpid=${secretWordsMap["###isd-500###"]}&corpsecret=${secretWordsMap["###isd-510###"]}"
        sh "curl -s -- \"${wechatRobotAccessTokenUrl}\" > tmp-WECHAT_ROBOT_ACCESS_TOKEN"
        def wechatRobotAccessToken = evaluate(readFile('tmp-WECHAT_ROBOT_ACCESS_TOKEN').trim().replaceAll("\\{", "[").replaceAll("\\}", "]"))["access_token"]
        sh 'rm -rf tmp-*'
        //发送微信消息通知
        if (wechatRobotAccessToken != null && wechatRobotAccessToken != '') {
            def wechatRobotUrl = "${WECHAT_ROBOT_URL}?access_token=${wechatRobotAccessToken}"
            sh "curl ${wechatRobotUrl} -H 'Content-Type:application/json' -X POST --data '${wechatMessage}'"
        }
    }
}