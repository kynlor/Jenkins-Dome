pipeline {
    agent any

    environment {
        REMOTE_HOST = '1.95.162.186'
        REMOTE_USER = 'root'
        REMOTE_CRED_ID = 'remote-server-ssh'
        REMOTE_DEPLOY_DIR = '/opt/app/web-demo'
        JAR_NAME = "web-demo-0.0.1-SNAPSHOT.jar"
        SERVER_PORT = '8080'
    }

    stages {
        stage('1.拉取代码') {
            steps {
                git url: 'git@github.com:kynlor/Jenkins-Dome.git',
                    branch: 'master',
                    credentialsId: 'github-ssh'
                echo "✅ 代码拉取完成"
            }
        }

        stage('2.代码测试') {
            steps {
                bat 'mvnw.cmd test'
                echo "✅ 单元测试通过"
            }
        }

        stage('3.Maven打包') {
            steps {
                bat 'mvnw.cmd clean package -DskipTests'
                echo "✅ 打包完成"
            }
        }

        stage('4.检查jar包') {
            steps {
                bat 'dir target\\*.jar /b'
                script {
                    def jarExists = fileExists("target/${JAR_NAME}")
                    if (!jarExists) {
                        error "❌ 找不到jar包 target/${JAR_NAME}，请检查JAR_NAME配置"
                    }
                }
            }
        }

        stage('5.SSH上传jar到服务器') {
            steps {
                script {
                    def remote = getRemote()
                    // sshPut 不会自动创建远程目录，先确保目录存在，并备份旧jar
                    sshCommand remote: remote,
                        command: """
                            mkdir -p ${REMOTE_DEPLOY_DIR}
                            cd ${REMOTE_DEPLOY_DIR}
                            if [ -f ${JAR_NAME} ]; then
                                mv ${JAR_NAME} ${JAR_NAME}.bak.\$(date +%Y%m%d%H%M%S)
                            fi
                        """
                    retry(3) {
                        sshPut remote: remote, from: "target/${JAR_NAME}", into: "${REMOTE_DEPLOY_DIR}/"
                    }
                    // 确认上传成功
                    sshCommand remote: remote,
                        command: "ls -lh ${REMOTE_DEPLOY_DIR}/${JAR_NAME}"
                }
                echo "✅ jar包上传完成"
            }
        }

        stage('6.远程部署服务') {
            steps {
                script {
                    def remote = getRemote()
                    sshCommand remote: remote,
                        command: """
                            cd ${REMOTE_DEPLOY_DIR}
                            # 优先用 pid 文件找旧进程；没有 pid 文件时用锚定到行首的方式匹配 java 进程，
                            # 避免 ps|grep 把当前 ssh 执行的 shell 自己也匹配进去导致误杀
                            PID=""
                            if [ -f app.pid ]; then
                                PID=\$(cat app.pid 2>/dev/null)
                            fi
                            if [ -z "\$PID" ] || ! kill -0 \$PID 2>/dev/null; then
                                PID=\$(pgrep -f "^java -jar ${JAR_NAME}" | head -1)
                            fi
                            if [ -n "\$PID" ]; then
                                echo "停止旧进程 PID: \$PID"
                                kill \$PID 2>/dev/null
                                sleep 3
                                kill -0 \$PID 2>/dev/null && kill -9 \$PID
                            fi
                            nohup java -jar ${JAR_NAME} --server.port=${SERVER_PORT} > app.log 2>&1 &
                            echo \$! > app.pid
                            echo "新服务已启动，端口 ${SERVER_PORT}，PID \$!"
                            sleep 5
                        """
                }
                echo "✅ 服务部署完成"
            }
        }

        stage('7.健康检查') {
            steps {
                script {
                    def remote = getRemote()
                    sshCommand remote: remote,
                        command: """
                            if ss -tulpn | grep :${SERVER_PORT} > /dev/null; then
                                echo "✅ 服务端口${SERVER_PORT}已启动，健康检查通过"
                            else
                                echo "❌ 服务启动失败，查看日志："
                                tail -20 ${REMOTE_DEPLOY_DIR}/app.log
                                exit 1
                            fi
                        """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 全流程CI/CD部署成功！访问地址：http://${REMOTE_HOST}:${SERVER_PORT}"
        }
        failure {
            echo "❌ 流水线失败，请查看各阶段日志排查"
        }
    }
}

// 抽取公共remote对象
def getRemote() {
    def remote = [:]
    remote.name = 'remote-server'
    remote.host = env.REMOTE_HOST
    remote.allowAnyHosts = true
    remote.retryCount = 3          // 连接失败自动重试3次
    remote.retryWaitSec = 5        // 每次重试间隔5秒
    remote.timeoutSec = 60         // SSH连接超时60秒

    // 注意：ssh-steps 插件不支持 remote.credentialsId（该字段会被忽略），
    // 必须先用 withCredentials 把凭据里的私钥取出来，再通过 remote.identity 传给 JSch。
    withCredentials([sshUserPrivateKey(
        credentialsId: env.REMOTE_CRED_ID,
        keyFileVariable: 'SSH_KEY_FILE',
        passphraseVariable: 'SSH_PASSPHRASE',
        usernameVariable: 'SSH_USERNAME'
    )]) {
        remote.user = env.SSH_USERNAME ?: env.REMOTE_USER
        // readFile 读出的是私钥内容（String），ssh-steps 会把它当作 identity 内容使用
        remote.identity = readFile(env.SSH_KEY_FILE)
        if (env.SSH_PASSPHRASE) {
            remote.passphrase = env.SSH_PASSPHRASE
        }
    }
    return remote
}
