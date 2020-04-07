pipeline {

    agent any

    tools {nodejs "node-13.12.0"}

    stages {
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
                archiveArtifacts artifacts: "${GIT_BRANCH}.tar.gz", fingerprint: true
            }
        }

        stage('Deploy - happy') {
            when {
                branch 'release-*'
            }

            steps {
                configFileProvider(
                    [configFile(fileId: 'lamboDev', variable: 'configFile')]) {
                    script {
                        def remoteEnvSet      = readProperties file: "$configFile"
                        env.awsSSHUser        = remoteEnvSet['awsSSHUser']
                        env.awsFrontEndHost   = remoteEnvSet['awsFrontEndHost']
                        env.credentialsID_SSH = remoteEnvSet['credentialsID_SSH']
                    }
                }

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
                tag pattern: '^happy-*', comparator: "REGEXP"
            }

            steps {
                sh 'echo "\${GIT_BRANCH}"'
            }
        }
    }

    post {
        always {
            sh 'echo stage end '
        }
    }
}
