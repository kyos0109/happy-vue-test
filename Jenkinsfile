@NonCPS
def getSendText() {
    return """
`Name: ` *${currentBuild.fullDisplayName}*
`Author: ` *${GIT_COMMITTER_EMAIL}*
`Status: ` *${currentBuild.result}*
`Duration: ` *${currentBuild.durationString}*
`URL: ` [Touch Me](${RUN_DISPLAY_URL})
    """
}

pipeline {

    agent any

    tools {nodejs "node-13.12.0"}

    stages {
        stage('Setting Info') {
            steps {
                script {
                    env.GIT_COMMITTER_EMAIL = sh(
                       script: "git --no-pager show -s --format='%ae' $GIT_COMMIT",
                       returnStdout: true
                    ).trim()
                }

                configFileProvider([configFile(fileId: 'lamboDev', variable: 'settingEnv')]) {
                    script {
                        def envItem = readProperties file: "$settingEnv"

                        envItem.each{ key, value -> 
                            env."${key}" = value
                        }
                    }
                }
            }
        }

        stage('Info') {
            steps {
                sh 'node --version'
                sh 'npm --version'
                sh 'yarn --version'
                sh 'printenv'
            }
        }

        stage('Setting') {
            steps {
                configFileProvider([configFile(fileId: env.GIT_BRANCH, variable: 'VERSION_SET')]) {
                    sh '''
                        cat ${VERSION_SET} > versionSet.json
                        CDN=`cat versionSet.json | awk -F ' ' '/ResourceCDN/{print $2}' | sed -E -e 's/"|,//g' | cut -d '/' -f3`
                        echo $CDN
                        sed -i.bak "s/CDNChangeMe/$CDN/g" public/index.html
                    '''
                }
            }
        }

        stage('Dependencies') {
            steps {
                sh 'yarn install'
            }
        }

        stage('Build') {
            steps {
                sh "yarn build --dest $GIT_BRANCH"
            }
        }

        stage('Artifacts') {
            steps {
                sh "ls -al"
                sh "mv versionSet.json ./$GIT_BRANCH/."
                sh "tar -czf ${GIT_BRANCH}.tar.gz ./$GIT_BRANCH"
                stash "${GIT_BRANCH}.tar.gz"
                archiveArtifacts artifacts: "${GIT_BRANCH}/**/*.*", fingerprint: true
            }
        }

        stage('Deploy - happy') {
            when {
                branch 'release-*'
            }

            steps {
                sshagent(credentials: [env.credentialsID_SSH]) {
                    sh '''
                        Domain=`echo "\${GIT_BRANCH}" | cut -d '-' -f3`
                        echo ${Domain}
                        ssh -o StrictHostKeyChecking=no "\${awsSSHUser}"@"\${awsFrontEndHost}" 'uptime'
                        tar -cz ./"\${GIT_BRANCH}" | ssh -o StrictHostKeyChecking=no "\${awsSSHUser}"@"\${awsFrontEndHost}" 'tar -xzf - -C ~/.'
                        ssh -o StrictHostKeyChecking=no "\${awsSSHUser}"@"\${awsFrontEndHost}" "sudo cp -a ~/"\${GIT_BRANCH}/." /usr/share/nginx/html/${Domain}"
                        ssh -o StrictHostKeyChecking=no "\${awsSSHUser}"@"\${awsFrontEndHost}" "sudo chown root: -R /usr/share/nginx/html/${Domain}"
                    '''
                }
            }
        }

        stage('Deploy - Production') {
            when {
                tag 'happy-*'
            }

            input {
                message "Deploy To Production?"
                ok "Yes"
                submitter "happy"
            }

            steps {
                sh 'echo "\${GIT_BRANCH}"'
            }
        }
    }

    post {
        always {
            sh 'printenv'
        }

        success {
            telegramSend(message: getSendText(), chatId: env.telegramChatId)
        }
        failure {
            telegramSend(message: getSendText(), chatId: env.telegramChatId)
        }
        unstable {
            telegramSend(message: getSendText(), chatId: env.telegramChatId)
        }
    }
}
