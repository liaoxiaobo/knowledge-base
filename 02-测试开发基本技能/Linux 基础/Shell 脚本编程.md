---
tags:
  - 实战
  - Shell
---
# Shell 脚本编程

> 来源：`source/_posts/linux-shell.md`  
> 整理日期：2026-06-16  > 适用场景：编写测试环境初始化脚本、批量执行用例、测试结果收集、CI/CD 前置/后置任务

---

## 一、Shell 脚本基础

Shell 是用户与 Linux 内核之间的翻译层。测试开发中，Shell 脚本常用于：

- 测试环境预检与初始化
- 批量执行测试用例并收集结果
- 日志备份与清理
- CI/CD Pipeline 中的前置/后置步骤

**执行方式差异**：

```bash
./script.sh          # 子 shell 中执行，变量不返回父 shell
sh script.sh         # 同上
source script.sh     # 当前 shell 中执行，变量保留
. script.sh          # 同 source
```

---

## 二、变量

### 2.1 定义与使用

```bash
NAME="tester"        # 等号两侧不能有空格
AGE=25

# 取值
echo $NAME
echo ${NAME}         # 推荐写法，避免歧义

# 命令结果赋值
TODAY=$(date +%Y-%m-%d)
TODAY=`date +%Y-%m-%d`  # 旧写法，不推荐
```

### 2.2 特殊变量

| 变量 | 含义 | 测试场景 |
|---|---|---|
| `$0` | 脚本名 | 记录执行入口 |
| `$1` ~ `$9` | 位置参数 | 接收环境名、版本号等参数 |
| `$#` | 参数个数 | 校验必填参数 |
| `$@` | 所有参数（独立引用） | 遍历参数 |
| `$?` | 上条命令退出码 | 判断命令是否成功 |
| `$$` | 当前脚本 PID | 写锁文件或日志 |

```bash
#!/bin/bash
ENV=$1
VERSION=$2

if [ $# -lt 2 ]; then
    echo "Usage: $0 <env> <version>"
    exit 1
fi

echo "Running tests on $ENV with version $VERSION"
```

### 2.3 环境变量

```bash
export PATH=$PATH:/opt/test-tools/bin
export TEST_ENV=dev
```

---

## 三、输入输出重定向

| 符号 | 含义 | 示例 |
|---|---|---|
| `>` | 标准输出覆盖写入 | `pytest > result.log` |
| `>>` | 标准输出追加写入 | `echo "done" >> result.log` |
| `2>` | 标准错误重定向 | `pytest 2> error.log` |
| `2>&1` | 标准错误合并到标准输出 | `pytest > run.log 2>&1` |
| `&>` | 标准输出和错误都重定向 | `pytest &> run.log` |
| `<` | 标准输入重定向 | `wc -l < file.txt` |
| `\|` | 管道 | `grep ERROR app.log \| wc -l` |

---

## 四、条件判断

### 4.1 文件判断

```bash
if [ -f "config.yaml" ]; then
    echo "配置文件存在"
fi

if [ -d "reports" ]; then
    echo "报告目录存在"
fi
```

| 标志 | 含义 |
|---|---|
| `-e` | 存在 |
| `-f` | 是普通文件 |
| `-d` | 是目录 |
| `-r` / `-w` / `-x` | 可读/可写/可执行 |
| `-s` | 文件非空 |

### 4.2 数值与字符串比较

```bash
COUNT=10
if [ $COUNT -gt 5 ]; then
    echo "数量大于 5"
fi

NAME="test"
if [ "$NAME" == "test" ]; then
    echo "匹配"
fi
```

| 数值 | 含义 | 字符串 | 含义 |
|---|---|---|---|
| `-eq` | 等于 | `==` | 等于 |
| `-ne` | 不等于 | `!=` | 不等于 |
| `-gt` | 大于 | `-z` | 为空 |
| `-lt` | 小于 | `-n` | 非空 |

### 4.3 逻辑组合

```bash
if [ -f "config.yaml" ] && [ "$ENV" == "prod" ]; then
    echo "生产环境配置已准备"
fi
```

---

## 五、流程控制

### 5.1 if-elif-else

```bash
if [ "$RESULT" == "PASS" ]; then
    echo "测试通过"
elif [ "$RESULT" == "FAIL" ]; then
    echo "测试失败"
else
    echo "结果未知"
fi
```

### 5.2 case

```bash
case $ENV in
    "dev")
        API_URL="http://dev.api.example.com"
        ;;
    "test")
        API_URL="http://test.api.example.com"
        ;;
    "prod")
        API_URL="http://api.example.com"
        ;;
    *)
        echo "未知环境: $ENV"
        exit 1
        ;;
esac
```

### 5.3 for 循环

```bash
for env in dev test prod; do
    echo "Deploying to $env"
done

# 遍历文件
for log in logs/*.log; do
    gzip "$log"
done
```

### 5.4 while 循环

```bash
# 逐行读取文件
while IFS= read -r line; do
    echo "$line"
done < cases.txt
```

---

## 六、函数

```bash
#!/bin/bash

log_info() {
    echo "[INFO] $(date '+%Y-%m-%d %H:%M:%S') $1"
}

run_tests() {
    local env=$1
    log_info "Starting tests on $env"
    pytest tests/ -m "$env"
    return $?
}

run_tests dev
```

**注意**：函数内变量默认是全局的，建议用 `local` 限制作用域。

---

## 七、常用测试场景脚本模板

### 7.1 环境预检脚本

```bash
#!/bin/bash
set -e  # 遇错即停

check_command() {
    if ! command -v "$1" &> /dev/null; then
        echo "缺少命令: $1"
        exit 1
    fi
}

check_command python3
check_command pytest
check_command docker

echo "环境检查通过"
```

### 7.2 批量执行并汇总结果

```bash
#!/bin/bash

REPORT_DIR="reports/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$REPORT_DIR"

for module in tests/module_*; do
    module_name=$(basename "$module")
    pytest "$module" --html="$REPORT_DIR/${module_name}.html" > "$REPORT_DIR/${module_name}.log" 2>&1
    echo "$module_name exit_code=$?" >> "$REPORT_DIR/summary.txt"
done

echo "报告目录: $REPORT_DIR"
```

### 7.3 清理过期测试数据

```bash
#!/bin/bash

find /var/log/test/ -name "*.log" -mtime +7 -delete
find /tmp/test_data/ -type d -empty -delete
```

---

## 八、脚本健壮性建议

1. **开头声明解释器和 `set -euo pipefail`**：
   ```bash
   #!/bin/bash
   set -euo pipefail
   ```
2. **变量引用加双引号**：避免路径含空格导致解析错误，`"$DIR"` 而非 `$DIR`。
3. **使用 `trap` 清理资源**：
   ```bash
   trap 'rm -rf "$TMP_DIR"' EXIT
   ```
4. **日志带时间戳**：便于与测试执行时间关联。
5. **参数校验**：必填参数缺失时给出用法提示并退出。
