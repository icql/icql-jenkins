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
                                    sh "echo \"* [[${GIT_AFTER.take(6)}]](https://gitee.com/icql/${GIT_REPONAME}/commit/${GIT_AFTER}) 新增分支${GIT_BRANCH}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS"
                                } else if (GIT_AFTER == empty40 || GIT_AFTER == empty8) {
                                    //删除分支
                                    sh "echo \"* [[${GIT_BEFORE.take(6)}]](https://gitee.com/icql/${GIT_REPONAME}/commit/${GIT_BEFORE}) 删除分支${GIT_BRANCH}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS"
                                } else {
                                    //普通提交
                                    sh "cd tmp-gitee-${GIT_REPONAME} && \
                                    (git log ${GIT_BRANCH} -5 --pretty=format:\"* [[%t]](https://gitee.com/icql/${GIT_REPONAME}/commit/%H) %s（%cn）\") > ../${GIT_REPONAME}_LATEST_COMMITS"
                                }
                            } else if (GIT_REF.contains('refs/tags/')) {
                                def GIT_TAG = GIT_REF.tokenize('/')[2]
                                if (GIT_BEFORE == empty40 || GIT_BEFORE == empty8) {
                                    //新增标签
                                    sh "echo \"* [[${GIT_AFTER.take(6)}]](https://gitee.com/icql/${GIT_REPONAME}/commit/${GIT_AFTER}) 新增标签${GIT_TAG}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS"
                                } else if (GIT_AFTER == empty40 || GIT_AFTER == empty8) {
                                    //删除标签
                                    sh "echo \"* [[${GIT_BEFORE.take(6)}]](https://gitee.com/icql/${GIT_REPONAME}/commit/${GIT_BEFORE}) 删除标签${GIT_TAG}（icql）\" > ${GIT_REPONAME}_LATEST_COMMITS"
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
    def latestCommits = ''
    if (fileExists("${GIT_REPONAME}_LATEST_COMMITS")) {
        latestCommits = readFile("${GIT_REPONAME}_LATEST_COMMITS").trim()
        sh "rm -rf ${GIT_REPONAME}_LATEST_COMMITS"
    }
    def message = "{\"msgtype\":\"markdown\",\"markdown\":{\"title\":\"SGG 通知\",\"text\":\"" +
            "### [${JOB_NAME}/${BUILD_NUMBER}](${BUILD_URL}/console) ${result}\\n" +
            "#### 最近同步的内容：\\n" +
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