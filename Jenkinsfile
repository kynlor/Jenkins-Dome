pipeline {
    agent any

    // 全局环境变量，后面直接改这里就行
    environment {
        // ========== 远程服务器配置 ==========
        REMOTE_HOST = '1.95.162.186'
        REMOTE_USER = 'root'                // 服务器用户名，一般是root
        REMOTE_CRED_ID = 'remote-server-ssh' // 后面要在Jenkins配的SSH凭证ID
        REMOTE_DEPLOY_DIR = '/opt/app/web-demo' // 服务器上的部署目录，自己建
        JAR_NAME = 'webdemo-0.0.1-SNAPSHOT.jar' // 你的jar包名，后面根据target里的实际名字改
        SERVER_PORT = '8080'               // 项目运行端口，和你SpringBoot配置一致
    }

    stages {
        // 1. 拉取GitHub代码
        stage('1.拉取代码') {
            steps {
                git url: 'git@github.com:kynlor/Jenkins-Dome.git',
                    branch: 'master',
                    credentialsId: 'github-ssh'
                echo "✅ 代码拉取完成"
            }
        }

        // 2. 单元测试
        stage('2.代码测试') {
            steps {
                bat 'mvnw.cmd test'
                echo "✅ 单元测试通过"
            }
        }

        // 3. Maven打包
        stage('3.Maven打包') {
            steps {
                bat 'mvnw.cmd clean package -DskipTests'
                echo "✅ 打包完成"
            }
        }

        // 4. 检查打包产物
        stage('4.检查jar包') {
            steps {
                bat 'dir target\\*.jar /b'
                script {
                    // 确认jar包存在
                    def jarExists = fileExists("target/${JAR_NAME}")
                    if (!jarExists) {
                        error "❌ 找不到jar包 target/${JAR_NAME}，请检查JAR_NAME配置"
                    }
                }
            }
        }

        // 5. 上传jar到远程服务器
        stage('5.SSH上传jar到服务器') {
            steps {
                // 用sshPut上传文件（需要SSH Pipeline Steps插件）
                sshPut remote: [
                    name: 'remote-server',
                    host: env.REMOTE_HOST,
                    username: env.REMOTE_USER,
                    credentialsId: env.REMOTE_CRED_ID
                ],
                from: "target/${JAR_NAME}",
                into: env.REMOTE_DEPLOY_DIR
                echo "✅ jar包上传完成"
            }
        }

        // 6. 远程部署执行脚本
        stage('6.远程部署服务') {
            steps {
                sshCommand remote: [
                    name: 'remote-server',
                    host: env.REMOTE_HOST,
                    username: env.REMOTE_USER,
                    credentialsId: env.REMOTE_CRED_ID
                ],
                command: """
                    # 进入部署目录
                    cd ${REMOTE_DEPLOY_DIR}

                    # 1. 停止旧服务（根据jar名找进程，不会误杀）
                    PID=\$(ps -ef | grep ${JAR_NAME} | grep -v grep | awk '{print \$2}')
                    if [ -n "\$PID" ]; then
                        echo "停止旧进程 PID: \$PID"
                        kill -9 \$PID
                        sleep 2
                    fi

                    # 2. 备份旧jar（可选，留最近3份）
                    if [ -f ${JAR_NAME} ]; then
                        mv ${JAR_NAME} ${JAR_NAME}.bak.\$(date +%Y%m%d%H%M)
                    fi

                    # 3. 后台启动新服务
                    nohup java -jar ${JAR_NAME} --server.port=${SERVER_PORT} > app.log 2>&1 &
                    echo "新服务已启动，端口 ${SERVER_PORT}"
                    sleep 5
                """
                echo "✅ 服务部署完成"
            }
        }

        // 7. 健康检查
        stage('7.健康检查') {
            steps {
                sshCommand remote: [
                    name: 'remote-server',
                    host: env.REMOTE_HOST,
                    username: env.REMOTE_USER,
                    credentialsId: env.REMOTE_CRED_ID
                ],
                command: """
                    # 检查服务端口是否监听
                    if netstat -tulpn | grep :${SERVER_PORT} > /dev/null; then
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

    post {
        success {
            echo "🎉 全流程CI/CD部署成功！访问地址：http://${REMOTE_HOST}:${SERVER_PORT}"
        }
        failure {
            echo "❌ 流水线失败，请查看各阶段日志排查"
        }
    }
}