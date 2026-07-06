---
title: "Playwright录制与提示词生成用例实战"
tags:
  - 项目
  - AI测试
  - Playwright
  - 提示词工程
  - 用例生成
created: "2026-07-06"
updated: "2026-07-06"
---

# 基于playwright 录制脚本+提示词的测试开发实践

## AI生成自动化用例步骤

### 3.1 录制用例脚本

工具相关文档：生成测试 | Playwright Python 中文网

录制脚本示例，脚本已包含元素定位+元素操作+测试数据

```bash
# 启动录制工具
playwright codegen --ignore-https-errors https://172.22.1.189:30000
```

```python
import re
from playwright.sync_api import Playwright, sync_playwright, expect


def run(playwright: Playwright) -> None:
    browser = playwright.chromium.launch(headless=False)
    context = browser.new_context(ignore_https_errors=True)
    page.goto("https://172.22.1.190:30000/ecs/#/ecs-instance-csv-list")
    page.get_by_text("弹性云服务器", exact=True).click()
    page.get_by_text("新建", exact=True).click()
    page.get_by_placeholder("请输入名称").first.click()
    page.get_by_placeholder("请输入名称").first.fill("test-demo")
    page.get_by_placeholder("请选择集群").click()
    page.locator("ul").filter(has_text=re.compile(r"^Autotest$")).locator("span").click()
    page.get_by_role("spinbutton").first.click()
    page.get_by_role("spinbutton").first.fill("1")
    page.get_by_text("选择计算规格").first.click()
    page.get_by_role("row", name="计算标准型 ecs.c6.large 2核 4GiB").get_by_role("radio").click()
    page.get_by_role("dialog").get_by_text("确定").click()
    page.get_by_placeholder("请选择", exact=True).nth(2).click()
    page.get_by_text("xstor-test").click()
    page.get_by_placeholder("请选择", exact=True).nth(3).click()
    page.locator("li").filter(has_text=re.compile(r"^镜像$")).click()
    page.get_by_placeholder("请选择操作系统版本").click()
    page.get_by_text("centos7.9").click()
    page.get_by_placeholder("请选择操作系统位数").click()
    page.get_by_text("64位").click()
    page.get_by_placeholder("请选择镜像").click()
    page.get_by_title("xstor-test").click()
    page.get_by_role("spinbutton").nth(1).click()
    page.get_by_role("spinbutton").nth(1).press("ArrowRight")
    page.get_by_role("spinbutton").nth(1).press("ArrowRight")
    page.get_by_role("spinbutton").nth(1).fill("50")
    page.get_by_role("textbox", name="请选择网络").click()
    page.get_by_text("Autotest", exact=True).nth(4).click()
    page.get_by_role("textbox", name="请选择子网").click()
    page.get_by_text("Autotest(10.151.146.0/24)").nth(2).click()
    page.get_by_placeholder("请输入密码").click()
    page.get_by_placeholder("请输入密码").fill("sugon@20")
    page.locator("div").filter(has_text=re.compile(r"^确认登录密码$")).get_by_role("textbox").click()
    page.locator("div").filter(has_text=re.compile(r"^确认登录密码$")).get_by_role("textbox").fill("sugon@20")
    page.get_by_placeholder("VNC密码最长为8位").click()
    page.get_by_placeholder("VNC密码最长为8位").fill("sugon@20")
    page.locator("div").filter(has_text=re.compile(r"^确认VNC密码$")).get_by_role("textbox").click()
    page.locator("div").filter(has_text=re.compile(r"^确认VNC密码$")).get_by_role("textbox").fill("sugon@20")
    page.get_by_text("立即创建").click()

    # ---------------------
    context.close()
    browser.close()


with sync_playwright() as playwright:
    run(playwright)
```

### 3.2 提示词设计

#### 通用提示词

```text
@README.md @sugon_web 请认真阅读框架源码和文档，根据最佳实践把上面playwright的录制脚本设计成一个符合项目风格的用例
```

#### 附加提示词（可选）

```text
该用例设计时还需要注意：
用例的前置数据：使用 fixture 函数 xxx 进行管理（参考conftest.py里的volume），让测试代码更加简洁
用例的断言：断言xx资源状态为"可用"、"挂载中"
用例的数据驱动：设计三组参数组合，其中节点数量的值分别是4、5，10，云硬盘类型的值为xstor-type、ceph-type
其它用例需求：.....
```

### 3.3 AI生成用例

录制脚本+用例提示词，一起发送给AI

Claude Sonnet 4模型生成的结果示例

```python
import re
from sugon_web.common.base import BasePage, submenu


class EcsPage(BasePage):

    def __init__(self, page, env):
        super().__init__(page, env)

    @submenu("弹性云服务器")
    def ecs_create(self, 
                   name,
                   cluster="Autotest",
                   count=1,
                   flavor="ecs.c6.large",
                   volume_type="xstor-test",
                   image_source="镜像",
                   os_version="centos7.9",
                   os_arch="64位",
                   image_name="xstor-test",
                   disk_size=50,
                   network="Autotest",
                   subnet="Autotest(10.161.43.0/24)",
                   ip_mode="自动分配",
                   login_password="sugon@20",
                   vnc_password="sugon@20"):
        """创建弹性云服务器"""
        self.btn_create.click()
        
        # 基本信息
        self._input_name.fill(name)
        self._select_cluster(cluster)
        self._input_count.fill(str(count))
        self._select_flavor(flavor)
        
        # 存储配置
        self._select_volume_type(volume_type)
        self._select_image_source(image_source)
        self._select_os_version(os_version)
        self._select_os_arch(os_arch)
        self._select_image(image_name)
        self._input_disk_size.fill(str(disk_size))
        
        # 网络配置
        self._select_network(network)
        self._select_subnet(subnet)
        self._select_ip_mode(ip_mode)
        
        # 登录配置
        self._input_login_password.fill(login_password)
        self._input_confirm_login_password.fill(login_password)
        self._input_vnc_password.fill(vnc_password)
        self._input_confirm_vnc_password.fill(vnc_password)
        
        # 提交创建
        self.btn_submit.click()

    # 私有方法 - 元素操作封装
    def _select_cluster(self, cluster):
        """选择集群"""
        self.get_by_placeholder("请选择集群").click()
        self.locator("ul").filter(has_text=re.compile(f"^{cluster}$")).locator("span").click()

    def _select_flavor(self, flavor):
        """选择计算规格"""
        self.get_by_text("选择计算规格").first.click()
        self.get_by_role("row", name=re.compile(flavor)).get_by_role("radio").click()
        self.get_by_role("dialog").get_by_text("确定").click()

    def _select_volume_type(self, volume_type):
        """选择存储类型"""
        self.get_by_placeholder("请选择", exact=True).nth(2).click()
        self.get_by_text(volume_type).click()

    def _select_image_source(self, source):
        """选择镜像来源"""
        self.get_by_placeholder("请选择", exact=True).nth(3).click()
        self.locator("li").filter(has_text=re.compile(f"^{source}$")).click()

    def _select_os_version(self, version):
        """选择操作系统版本"""
        self.get_by_placeholder("请选择操作系统版本").click()
        self.get_by_text(version).click()

    def _select_os_arch(self, arch):
        """选择操作系统位数"""
        self.get_by_placeholder("请选择操作系统位数").click()
        self.get_by_text(arch).click()

    def _select_image(self, image_name):
        """选择镜像"""
        self.get_by_placeholder("请选择镜像").click()
        self.get_by_title(image_name).click()

    def _select_network(self, network):
        """选择网络"""
        self.get_by_role("textbox", name="请选择网络").click()
        self.get_by_text(network, exact=True).nth(4).click()

    def _select_subnet(self, subnet):
        """选择子网"""
        self.get_by_role("textbox", name="请选择子网").click()
        self.get_by_text(subnet).nth(2).click()

    def _select_ip_mode(self, mode):
        """选择IP分配模式"""
        self.get_by_role("textbox", name="请选择分配模式").click()
        self.get_by_text(mode).nth(2).click()

    # 页面元素属性
    @property
    def _input_name(self):
        """云服务器名称输入框"""
        return self.get_by_placeholder("请输入名称").first

    @property
    def _input_count(self):
        """数量输入框"""
        return self.get_by_role("spinbutton").first

    @property
    def _input_disk_size(self):
        """磁盘大小输入框"""
        return self.get_by_role("spinbutton").nth(1)

    @property
    def _input_login_password(self):
        """登录密码输入框"""
        return self.get_by_placeholder("请输入密码")

    @property
    def _input_confirm_login_password(self):
        """确认登录密码输入框"""
        return self.locator("div").filter(has_text=re.compile(r"^确认登录密码$")).get_by_role("textbox")

    @property
    def _input_vnc_password(self):
        """VNC密码输入框"""
        return self.get_by_placeholder("VNC密码最长为8位")

    @property
    def _input_confirm_vnc_password(self):
        """确认VNC密码输入框"""
        return self.locator("div").filter(has_text=re.compile(r"^确认VNC密码$")).get_by_role("textbox")

    @property
    def btn_submit(self):
        """立即创建按钮"""
        return self.get_by_text("立即创建")
```

## 最佳实践

### 4.1 用例录制

优先使用测试环境进行录制，自动化测试所依赖的基础数据已具备

不操作与用例步骤无关的页面元素，避免影响到AI生成用例的效果

创建资源类的用例，需要把删除资源的操作步骤一起录制

非创建资源的用例，可以只录制操作本身即可（如修改资源名称），通过提示词告知AI测试用例所需的前置数据

### 4.2 用例参数化的使用

参数化的用途：测试多种参数组合，适用于创建类场景

参数化的判断标准：

- 核心业务逻辑相关的参数
- 容易出现兼容性问题的参数
- 不同值会产生明显不同行为的参数

不参数化的判断标准：

- 环境固定的配置参数
- 很少变化的默认值参数
- 对测试结果影响很小的参数
- 维护成本高但测试价值低的参数

### 4.3 POM设计模式

#### 页面封装

继承 BasePage 类，BasePage 提供了统一的基础能力(公共元素定位、公共断言)

每个页面都有对应的类（LoginPage, EVSPage）,类名语义明确

页面封装包括对元素定位和业务操作的封装

#### 元素定位

使用 @property 装饰器封装元素定位，返回 Locator 对象

尽量使用语义化的定位方式（get_by_placeholder, get_by_text）

#### 业务操作封装

对外只暴露具体的业务操作方法，如login() 方法封装了完整的登录流程
