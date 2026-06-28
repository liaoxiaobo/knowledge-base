---
tags:
  - "#实战"
  - "#自动化框架"
  - "#RobotFramework"
---


# Robot Framework

> 摘要：Robot Framework 是基于关键字驱动的自动化测试框架，适合业务测试人员与开发协同编写验收测试。本文从测试开发视角整理其核心概念、分层设计方法，以及配置管理、测试库封装、CI/CD 集成等项目级实践经验。

## 一、为什么测试开发需要 Robot Framework

Robot Framework（以下简称 RF）是 Python 生态中成熟的关键字驱动测试框架，它的特点决定了适用场景：

- **关键字驱动**：测试用例由可复用的关键字组成，可读性接近自然语言，便于业务测试人员参与维护。
- **分层解耦**：测试用例、业务关键字、底层测试库职责清晰，适合大型项目的长期维护。
- **多协议支持**：通过扩展库可覆盖 Web、API、数据库、SSH、移动端等多种测试类型。
- **报告内置**：自动生成 `log.html`、`report.html`、`output.xml`，无需额外配置。
- **CI/CD 友好**：命令行驱动，退出码明确，易于接入 Jenkins、GitLab CI 等流水线。

不过，RF 更适合**业务规则清晰、用例数量大、需要非开发人员参与维护**的场景。如果团队以 Python 开发为主、用例变动频繁且追求灵活性，Pytest 通常是更好的选择。

## 二、核心概念

| 概念 | 说明 | 类比 Pytest |
|---|---|---|
| **Test Case** | 测试用例，由关键字和数据组成 | `test_` 函数 |
| **Keyword** | 可复用的操作单元，分为用户关键字和库关键字 | fixture / helper 函数 |
| **Test Suite** | 测试用例文件（`.robot`）的集合 | 测试模块/目录 |
| **Resource** | 用户关键字和资源文件（`.robot`），供多个 Suite 引用 | 公共 helper 模块 |
| **Library** | Python/Java 编写的测试库，提供底层能力 | 自定义 Python 包 |
| **Variable** | 变量，支持标量、列表、字典 | 常量 / fixture 返回值 |
| **Tag** | 用例标签，用于筛选、统计、报告分类 | `pytest.mark` |

## 三、分层设计

RF 项目推荐采用三层结构，把「业务描述」和「技术实现」解耦：

```text
project/
├── Suites/              # 测试用例层：只描述业务操作和断言
│   ├── compute/
│   │   └── test_ecs.robot
│   └── network/
│       └── test_vpc.robot
├── Resource/            # 业务关键字层：封装业务操作
│   ├── Common.robot
│   ├── Compute.robot
│   └── Network.robot
└── Library/             # 底层测试库层：Python 实现技术细节
    ├── Basedata.py
    ├── Rest.py
    └── ServiceAdd.py
```

### 3.1 Suites 测试用例层

用例层只关心「做什么」，不关心「怎么做」：

```robot
*** Settings ***
Resource    ../Resource/Compute.robot

*** Test Cases ***
创建云服务器
    [Tags]    smoke    compute
    登录控制台
    进入云服务器列表
    点击创建云服务器
    填写基础配置    name=test-ecs    flavor=2C4G
    提交创建
    校验创建结果    运行中
```

### 3.2 Resource 业务关键字层

Resource 文件封装业务操作，由多个用例共享：

```robot
*** Settings ***
Library    ../Library/Rest.py
Library    ../Library/Basedata.py

*** Keywords ***
登录控制台
    ${base_url}    Get Baseurl
    Open Browser    ${base_url}    chrome
    Input Text    id=username    ${ADMIN_USER}
    Input Password    id=password    ${ADMIN_PASS}
    Click Button    id=login

进入云服务器列表
    Click Element    link=弹性云服务器
    Wait Until Page Contains    实例

填写基础配置
    [Arguments]    ${name}    ${flavor}
    Input Text    id=name    ${name}
    Select From List By Label    id=flavor    ${flavor}
```

### 3.3 Library 底层测试库层

Library 用 Python 实现底层能力，例如配置读取、REST 调用、环境准备：

```python
# Library/Rest.py
from robot.api.deco import keyword
from robot.libraries.BuiltIn import BuiltIn

class Rest:
    @keyword("Get Baseurl")
    def get_baseurl(self):
        env = BuiltIn().get_variable_value("${ENV}")
        config = load_config(f"config/{env}.yaml")
        BuiltIn().set_global_variable("${BASE_URL}", config["base_url"])
        return config["base_url"]
```

## 四、配置与测试数据管理

### 4.1 环境配置分离

将环境相关配置放到 YAML/INI 文件中，通过变量注入：

```yaml
# config/test.yaml
base_url: https://test.example.com
admin_user: admin
admin_pass: admin123
storage_pool: xstor-test
```

```robot
*** Variables ***
${ENV}    test

*** Settings ***
Variables    ../config/${ENV}.yaml

*** Test Cases ***
示例用例
    Log    ${BASE_URL}
```

### 4.2 敏感信息处理

密码、Token 等敏感信息不要硬编码到用例或配置文件中：

- 通过环境变量注入：`--variable PASS:${PASS}`
- 或在 CI/CD 中使用凭据管理工具（Jenkins Credentials、GitHub Secrets）。

### 4.3 动态数据生成

在 Library 中封装数据生成逻辑，避免用例层硬编码：

```python
# Library/Basedata.py
import random
from robot.api.deco import keyword

class Basedata:
    @keyword("Generate Random Name")
    def generate_random_name(self, prefix="res"):
        return f"{prefix}-{random.randint(1000, 9999)}"
```

## 五、测试库封装建议

### 5.1 命名规范

| 层级 | 命名建议 | 示例 |
|---|---|---|
| 用户关键字 | 动宾结构，中文或英文 | `登录控制台`、`Create ECS` |
| 库关键字 | 驼峰命名，语义清晰 | `Get Baseurl`、`Generate Random Name` |
| 变量 | 大写，下划线分隔 | `${ADMIN_USER}`、`${BASE_URL}` |
| 文件 | 按业务域组织 | `Compute.robot`、`Rest.py` |

### 5.2 错误处理与日志

- 在 Library 中使用 `logger` 输出关键步骤，便于失败时定位。
- 对不稳定操作（如页面加载、异步接口）添加显式等待或重试。
- 避免在 Library 中直接断言，断言应留在用例层或 Resource 层。

## 六、命令行执行与报告

### 6.1 基本执行

```bash
# 执行单个用例文件
robot Suites/compute/test_ecs.robot

# 执行整个目录
robot Suites/

# 按标签筛选
robot -i smoke Suites/
robot -e unstable Suites/

# 指定变量
robot -v ENV:staging -v BROWSER:firefox Suites/

# 指定输出目录
robot -d results Suites/
```

### 6.2 输出文件

执行后默认生成：

- `output.xml`：机器可读的完整结果，供 Jenkins 等工具解析。
- `log.html`：详细日志，包含每步输入输出。
- `report.html`：汇总报告，包含统计和趋势。

### 6.3 与 Jenkins 集成要点

详细 Jenkins 配置参见 `3.4.1 Jenkins Pipeline`，RF 相关要点：

- 安装 `robot` 插件以解析 `output.xml`。
- Pipeline 中使用 `robot` 命令执行测试。
- 通过 `passThreshold` / `unstableThreshold` 设置质量门禁。
- 邮件/飞书通知中附带 `report.html` 链接。

```groovy
stage('API Test') {
    steps {
        sh 'docker run --rm -v jenkins-data:/robot-results ${IMAGE} robot --outputdir /robot-results /project/Suites'
    }
}
post {
    always {
        robot logFileName: 'log.html', outputFileName: 'output.xml',
              outputPath: '/var/jenkins_home/', passThreshold: 100.0,
              reportFileName: 'report.html', unstableThreshold: 90.0
    }
}
```

## 七、项目级 Checklist

- [ ] 用例层只描述业务步骤，不直接调用底层 API。
- [ ] Resource 层按业务域拆分，避免单文件过大。
- [ ] Library 层用 Python 实现技术细节，保持关键字语义清晰。
- [ ] 环境配置、测试数据、敏感信息分离管理。
- [ ] 自定义关键字命名统一，便于业务测试人员理解。
- [ ] 标签体系完整（如 `smoke`、`regression`、`unstable`），支持按场景筛选。
- [ ] `output.xml` 被 CI/CD 正确解析，作为质量门禁依据。
- [ ] 报告输出目录规范，避免不同 Job 之间互相覆盖。

## 八、RF 与 Pytest 的选型建议

| 维度 | Robot Framework | Pytest |
|---|---|---|
| 用例可读性 | 高，接近自然语言 | 中，依赖代码能力 |
| 非开发人员参与 | 容易 | 较难 |
| 灵活性 | 中，受关键字结构约束 | 高，纯 Python |
| 生态与插件 | 丰富但偏测试领域 | 极丰富，通用开发也可用 |
| 复杂数据构造 | 一般 | 强 |
| 大规模并行 | 需结合外部工具 | pytest-xdist 原生支持 |
| 推荐场景 |  acceptance 测试、业务回归 | 单元/集成/API/UI 自动化、开发驱动 |

## 参考资源

- [Robot Framework 官方文档](https://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html)
- [Robot Framework 开源库索引](https://robotframework.org/#resources)
- [SeleniumLibrary 文档](https://github.com/robotframework/SeleniumLibrary)
- [RequestsLibrary 文档](https://github.com/MarketSquare/robotframework-requests)