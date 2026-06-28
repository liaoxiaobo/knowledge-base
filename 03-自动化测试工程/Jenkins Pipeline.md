---
tags:
  - 实战
  - Jenkins
  - CI/CD
---


# Jenkins Pipeline

> 摘要：Jenkins Pipeline 是实现自动化测试持续集成的核心载体。本文整理 Jenkins 在自动化测试场景下的部署、Pipeline 设计、触发策略、报告集成与通知机制，帮助将接口/UI 自动化测试稳定接入 CI/CD 流程。

## 一、为什么自动化测试需要 Jenkins Pipeline

自动化测试的价值只有在持续运行中才能体现。Jenkins Pipeline 提供了：

- **自动化触发**：代码提交、定时任务、手动触发、上游 Job 联动。
- **环境一致性**：通过 Docker 镜像封装测试运行环境，避免"本地能跑、CI 不能跑"。
- **结果可视化**：集成 Allure、Robot Framework 等报告插件，自动归档历史结果。
- **失败通知**：通过邮件、飞书、Slack 等渠道及时推送失败信息。
- **质量门禁**：根据通过率或失败数量决定是否阻断后续流程。

## 二、Jenkins 部署方式

### 2.1 Docker 部署（推荐）

使用 Docker 运行 Jenkins 可以快速搭建，且便于迁移和备份：

```bash
docker run -u root -d \
  --name jenkins-ci \
  -p 8081:8080 \
  -p 50000:50000 \
  -v /etc/localtime:/etc/localtime \
  -v /etc/timezone:/etc/timezone \
  -v jenkins-data:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  jenkinsci/blueocean
```

关键点：

- 挂载 `docker.sock` 和 `docker` 二进制文件，实现 DooD（Docker-outside-of-Docker），让 Jenkins 容器内可以调用宿主机 Docker。
- `jenkins-data` 卷持久化配置、插件、Job 历史。
- 首次启动需要解锁并安装推荐插件。

### 2.2 必要插件

| 插件 | 用途 |
|---|---|
| Pipeline | 支持 Jenkinsfile |
| Git / GitHub / GitLab | 源码拉取 |
| Allure Jenkins Plugin | 解析 Allure 报告 |
| robot | 解析 Robot Framework `output.xml` |
| Email Extension | 邮件通知 |
| Blue Ocean | 更直观的 Pipeline 可视化 |

## 三、自动化测试 Pipeline 结构

一个典型的自动化测试 Pipeline 分为四个阶段：

```text
Pipeline
├── 1. 环境准备：拉取代码、构建测试镜像
├── 2. 测试执行：运行接口/UI/RF 测试
├── 3. 报告归档：收集 Allure / RF / pytest-html 报告
└── 4. 结果通知：邮件/飞书推送，质量门禁判断
```

## 四、Pipeline 脚本详解

### 4.1 基于 Docker 的 Pytest Pipeline

```groovy
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        IMAGE_NAME = "auto-test:${BUILD_NUMBER}"
        ALLURE_RESULTS = "allure-results"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${GIT_BRANCH}",
                    credentialsId: "${GIT_CREDENTIALS}",
                    url: "${GIT_REPO_URL}"
            }
        }

        stage('Build Test Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME} .'
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    docker run --rm \
                        -v ${WORKSPACE}/${ALLURE_RESULTS}:/app/${ALLURE_RESULTS} \
                        -e TEST_ENV=${TEST_ENV} \
                        ${IMAGE_NAME} \
                        pytest tests -v --alluredir ./${ALLURE_RESULTS}
                '''
            }
        }
    }

    post {
        always {
            allure includeProperties: false,
                   jdk: '',
                   results: [[path: "${ALLURE_RESULTS}"]]
        }
        success {
            echo '测试通过'
        }
        failure {
            echo '测试失败，请查看 Allure 报告'
        }
    }
}
```

### 4.2 基于 Docker 的 Robot Framework Pipeline

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = "rf-test:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${GIT_BRANCH}",
                    credentialsId: "${GIT_CREDENTIALS}",
                    url: "${GIT_REPO_URL}"
            }
        }

        stage('Build Test Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME} .'
            }
        }

        stage('Run RF Tests') {
            steps {
                sh '''
                    docker run --rm \
                        -v ${WORKSPACE}/robot-results:/robot-results \
                        ${IMAGE_NAME} \
                        robot --outputdir /robot-results /project/Suites
                '''
            }
        }
    }

    post {
        always {
            robot logFileName: 'log.html',
                  outputFileName: 'output.xml',
                  outputPath: "${WORKSPACE}/robot-results",
                  passThreshold: 100.0,
                  reportFileName: 'report.html',
                  unstableThreshold: 90.0
        }
        success {
            mail to: "${RECIPIENTS}",
                 subject: "${JOB_NAME} 测试通过",
                 body: "测试版本分支：${GIT_BRANCH}\n报告地址：${BUILD_URL}"
        }
        failure {
            mail to: "${RECIPIENTS}",
                 subject: "${JOB_NAME} 测试失败",
                 body: "测试版本分支：${GIT_BRANCH}\n报告地址：${BUILD_URL}"
        }
    }
}
```

### 4.3 测试镜像 Dockerfile 示例

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["pytest"]
```

## 五、触发策略

### 5.1 Webhook 触发

代码提交后自动触发测试：

1. Jenkins Job 中配置「构建触发器 → 触发远程构建」，设置 Token。
2. 在 GitLab/GitHub 仓库 Webhook 中填写：`JENKINS_URL/job/job-name/build?token=TOKEN_NAME`。
3. 如需安全校验，配置 Secret Token 或 Jenkins 的 GitLab/GitHub 插件。

### 5.2 定时触发

适合夜间回归：

```groovy
pipeline {
    triggers {
        cron('H 2 * * *')  // 每天凌晨 2 点左右执行
    }
}
```

### 5.3 上游 Job 触发

适合在环境部署完成后自动触发测试：

```groovy
pipeline {
    triggers {
        upstream(upstreamProjects: 'deploy-env', threshold: hudson.model.Result.SUCCESS)
    }
}
```

## 六、测试报告集成

### 6.1 Allure 报告

Pytest 项目常用 Allure：

- 执行时生成 `allure-results` 目录。
- Pipeline `post` 阶段调用 `allure` 步骤解析并展示。
- 注意每次运行前清空旧数据，避免历史结果干扰。

### 6.2 Robot Framework 报告

RF 项目原生生成 `output.xml`、`log.html`、`report.html`：

- 使用 `robot` 插件解析 `output.xml`。
- 在 Job 首页展示通过率、失败用例数。
- 设置 `passThreshold` 和 `unstableThreshold` 作为质量门禁。

### 6.3 pytest-html 报告

轻量场景可直接生成 HTML：

```bash
pytest --html=report.html --self-contained-html
```

然后在 Pipeline 中使用 `publishHTML` 归档：

```groovy
post {
    always {
        publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: '.',
            reportFiles: 'report.html',
            reportName: 'Test Report'
        ])
    }
}
```

## 七、通知机制

### 7.1 邮件通知

```groovy
post {
    failure {
        mail to: "${RECIPIENTS}",
             subject: "${JOB_NAME} 测试失败 - Build #${BUILD_NUMBER}",
             body: """
                自动化测试未通过，请相关同事分析定位。
                项目：${JOB_NAME}
                分支：${GIT_BRANCH}
                构建号：${BUILD_NUMBER}
                报告地址：${BUILD_URL}
             """
    }
}
```

### 7.2 飞书/钉钉/Slack 通知

通过 HTTP 请求推送飞书机器人：

```groovy
post {
    always {
        script {
            def status = currentBuild.currentResult
            def color = status == 'SUCCESS' ? 'green' : 'red'
            sh """
                curl -X POST ${FEISHU_WEBHOOK} \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msg_type": "text",
                        "content": {
                            "text": "${JOB_NAME} 构建${status}\n分支：${GIT_BRANCH}\n报告：${BUILD_URL}"
                        }
                    }'
            """
        }
    }
}
```

## 八、多环境管理

同一套测试代码需要在测试环境、预发布环境、生产环境运行，建议：

- 环境配置放在 `config/test.yaml`、`config/staging.yaml` 中。
- Jenkins 中通过参数化构建选择环境：

```groovy
parameters {
    choice(name: 'TEST_ENV', choices: ['test', 'staging'], description: '测试环境')
}
```

- Pipeline 中将环境变量传入容器：

```groovy
sh '''
    docker run --rm -e TEST_ENV=${TEST_ENV} ${IMAGE_NAME} pytest tests
'''
```

## 九、项目级 Checklist

- [ ] Jenkins 使用 Docker 部署，数据和插件持久化。
- [ ] 安装必要的插件：Pipeline、Git、Allure/robot、Email Extension。
- [ ] Pipeline 中使用 Docker 镜像运行测试，保证环境一致性。
- [ ] 源码拉取使用 Credentials，避免明文密码。
- [ ] 测试执行阶段设置超时，防止 Job 挂死。
- [ ] 报告数据挂载到宿主机，便于 Jenkins 解析和归档。
- [ ] 配置 Webhook/定时/上游触发，覆盖日常开发和夜间回归。
- [ ] 设置质量门禁（通过率阈值），失败时阻断后续流程或发送告警。
- [ ] 通知信息包含分支、构建号、报告链接，方便快速定位。
- [ ] 定期清理历史构建，避免磁盘占满。

## 十、可复用模板

### 10.1 最小可用 Pytest Pipeline

```groovy
pipeline {
    agent any

    environment {
        TEST_ENV = "${params.TEST_ENV ?: 'test'}"
    }

    stages {
        stage('Test') {
            steps {
                sh 'pytest tests -v --alluredir=allure-results'
            }
        }
    }

    post {
        always {
            allure results: [[path: 'allure-results']]
        }
    }
}
```

### 10.2 参数化构建配置

```groovy
parameters {
    choice(name: 'TEST_ENV', choices: ['test', 'staging'], description: '目标环境')
    string(name: 'GIT_BRANCH', defaultValue: 'main', description: '测试分支')
    booleanParam(name: 'RUN_SMOKE', defaultValue: true, description: '是否执行冒烟测试')
}
```

## 参考资源

- [Jenkins 官方文档](https://www.jenkins.io/doc/)
- [Jenkins Pipeline 语法](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Allure Jenkins Plugin](https://plugins.jenkins.io/allure-jenkins-plugin/)
- [Robot Framework Jenkins Plugin](https://plugins.jenkins.io/robot/)