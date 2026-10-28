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
                    sshPut remote: remote, from: "target/${JAR_NAME}", into: env.REMOTE_DEPLOY_DIR
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
                            PID=\$(ps -ef | grep ${JAR_NAME} | grep -v grep | awk '{print \$2}')
                            if [ -n "\$PID" ]; then
                                echo "停止旧进程 PID: \$PID"
                                kill -9 \$PID
                                sleep 2
                            fi
                            if [ -f ${JAR_NAME} ]; then
                                mv ${JAR_NAME} ${JAR_NAME}.bak.\$(date +%Y%m%d%H%M)
                            fi
                            nohup java -jar ${JAR_NAME} --server.port=${SERVER_PORT} > app.log 2>&1 &
                            echo "新服务已启动，端口 ${SERVER_PORT}"
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
    remote.user = env.REMOTE_USER
    remote.credentialsId = env.REMOTE_CRED_ID
    remote.allowAnyHosts = true
    return remote
}
