# 轻点书城 · 接口自动化测试框架

基于 **Python + pytest + YAML 数据驱动 + Allure 报告** 的接口自动化测试框架，覆盖单接口测试与业务场景（流程）测试，支持参数提取传递、数据库断言、钉钉结果通知等功能。

## 技术栈

| 类别 | 技术 |
| --- | --- |
| 语言 / 运行环境 | Python 3.x |
| 测试框架 | pytest（参数化、fixture、用例排序） |
| 用例格式 | YAML 数据驱动 |
| HTTP 请求 | requests |
| 测试报告 | Allure / pytest-tmreport（HTML） |
| 断言 | 字符串包含、相等、不相等、任意值、SQL 数据库断言 |
| 数据库 | MySQL、Redis、MongoDB、ClickHouse、Oracle |
| 其他 | 钉钉机器人通知、邮件、Jenkins、PyQt5 用例生成工具 |

## 目录结构

```
轻点书城/
├── base/                  # 基础封装：接口请求、业务场景请求、用例生成工具等
│   ├── apiutil.py         # 单接口请求封装（RequestBase）
│   ├── apiutil_business.py# 业务场景（多接口串联）请求封装
│   ├── generateId.py      # Allure feature/story 编号生成
│   └── new_testcase_tools.py  # PyQt5 可视化用例 yaml 生成工具
├── common/                # 公共方法封装
│   ├── sendrequest.py     # requests 请求封装
│   ├── assertions.py      # 断言封装（contains/eq/ne/rv/db）
│   ├── readyaml.py        # yaml 用例读取、extract 数据读写
│   ├── debugtalk.py       # 用例中 ${函数名()} 的具体实现
│   ├── connection.py      # 各类数据库连接
│   ├── recordlog.py       # 日志记录
│   ├── dingRobot.py       # 钉钉机器人通知
│   ├── semail.py          # 邮件发送
│   ├── Pjenkins.py        # Jenkins 调用
│   ├── handleExcel.py / operationcsv.py / operxml.py  # 数据文件读写
│   └── two_dimension_data.py  # 终端表格打印
├── conf/                  # 全局配置目录
│   ├── config.ini         # 环境配置：接口 host、数据库、邮件、SSH 等
│   ├── setting.py         # 全局设置：日志级别、超时、报告类型、文件路径、钉钉开关
│   └── operationConfig.py # 读取 config.ini
├── data/                  # 测试数据（登录用例、csv、excel、SQL xml 模板）
├── testcase/              # 测试用例目录
│   ├── Single interface/  # 单接口用例（增删改查）
│   ├── ProductManager/    # 商品/下单/支付等接口用例
│   └── Business interface/# 业务场景用例（登录→下单→支付 串联）
├── logs/                  # 运行日志（已 gitignore）
├── report/                # 测试报告产物（已 gitignore）
├── conftest.py            # 全局 fixture：清理 extract、登录、结果摘要与钉钉通知
├── pytest.ini             # pytest 规则约束
├── extract.yaml           # 接口间参数提取的临时存放文件
├── environment.xml        # Allure 报告「环境」页展示内容
├── requirements.txt       # 三方依赖
└── run.py                 # 主程序入口
```

## 快速开始

### 1. 安装依赖

```bash
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple/
```

安装 Allure 命令行工具（用于生成/查看报告），并保证 `allure` 已加入 PATH：
Allure 安装说明：https://docs.qameta.io/allure/

### 2. 配置环境

- `conf/config.ini` 的 `[api_envi]` 节配置被测系统地址：

  ```ini
  [api_envi]
  host = http://127.0.0.1:8787
  ```

  数据库、邮件、SSH 等连接信息同样在该文件中配置（请勿提交真实敏感信息）。

- `conf/setting.py` 中可调整：

  ```python
  LOG_LEVEL = logging.DEBUG      # 日志级别
  API_TIMEOUT = 60                # 接口超时（秒）
  REPORT_TYPE = 'allure'          # 报告类型：allure / tm
  dd_msg = True                   # 是否发送钉钉结果通知
  ```

### 3. 运行测试

```bash
# 方式一：主程序入口（推荐）
python run.py

# 方式二：直接执行 pytest
pytest -vs ./testcase
```

- `REPORT_TYPE = 'allure'` 时：执行完毕自动复制 `environment.xml` 到报告原始数据目录并执行 `allure serve ./report/temp` 打开报告。
- `REPORT_TYPE = 'tm'` 时：生成 HTML 报告并自动在浏览器打开 `report/tmreport/testReport.html`。

## 用例编写规范

用例由 `baseInfo`（接口基础信息）和 `testCase`（用例列表）两部分组成，**两者关键字均不可缺少**。

```yaml
- baseInfo:
    api_name: 新增用户
    url: /dar/user/addUser        # 只写接口路径，host 取自 conf/config.ini
    method: POST
    header:
      Content-Type: application/x-www-form-urlencoded;charset=UTF-8
      token: ${get_extract_data(token)}
    cookies:                       # 可选，视项目是否需要 cookie
      SESSION: ${get_extract_data(Cookie,SESSION)}
  testCase:
    - case_name: 正确新增用户
      data:                        # params / data / json 三选一
        username: testadduser
        password: test6789890
        role_id: 123456789
      files:                       # 可选，文件上传
        file: ./data/heimingdan.xlsx
      validation:                  # 断言，contains 建议写在最前
        - contains: {status_code: 200}
        - contains: {'msg': '新增成功'}
        - eq: {'state': '已入网'}          # 相等
        - ne: {'state': '已入网'}          # 不相等
        - rv: {"data": 2}                  # 返回值任意位置匹配
        - db: select * from sys_user where login_name='test999'   # SQL 断言
      extract:                     # 提取单个参数，支持 jsonpath 与正则
        id: $.data
        status: '"status":"(.*?)"'
      extract_list:                # 提取多个参数，以列表返回
        id: $.result.id
```

### 参数传递与函数

- 取值格式：`${函数名(*args)}`，函数在 `common/debugtalk.py` 中实现，例如 `${get_extract_data(token)}`、`${start_time()}`、`${md5_encryption(abc)}`。
- `get_extract_data(node, randoms)`：从 `extract.yaml` 读取，`randoms` 支持 `0` 随机、`1/2/...` 顺序取值、`-1` 逗号拼接、`-2` 转列表。
- 接口间依赖：上一个接口用 `extract` 写入 `extract.yaml`，下一个接口用 `${get_extract_data(key)}` 取出；每次运行前 `conftest.py` 会自动清空该文件。

### 请求参数类型

| 场景 | data 类型 | Content-Type |
| --- | --- | --- |
| POST 表单 | `data` | `application/x-www-form-urlencoded;charset=UTF-8` |
| POST JSON | `json` | `application/json;charset=UTF-8` |
| GET 传参 | `params` | — |
| 文件上传 | `files` | `multipart/form-data; charset=utf-8` |

jsonpath 在线解析：http://www.atoolbox.net/Tool.php?Id=792

### 用例分类

| 类型 | 说明 |
| --- | --- |
| 单接口测试 | `testcase/Single interface/`，使用 `base/apiutil.py` 的 `RequestBase`，一个 yaml 中多条 `testCase` 通过 `@pytest.mark.parametrize` 数据驱动 |
| 业务场景测试 | `testcase/Business interface/`，使用 `base/apiutil_business.py`，一个 yaml 内多个接口按顺序串联执行，模拟完整业务流程 |

测试类与方法通过 `@allure.feature` / `@allure.story` 组织报告目录，`@pytest.mark.run(order=N)` 控制执行顺序。

## 测试报告与通知

- **Allure 报告**：`run.py` 执行后自动打开；也可手动执行 `allure serve ./report/temp`。报告「环境」页内容来自根目录 `environment.xml`（该文件会被 `--clean-alluredir` 清空，故框架会在执行前复制进报告目录）。
- **HTML 报告**：`conf/setting.py` 中 `REPORT_TYPE = 'tm'`。
- **钉钉通知**：`dd_msg = True` 时，`conftest.py` 的 `pytest_terminal_summary` 会收集通过/失败/错误/跳过数量与耗时并推送钉钉群（配置见 `common/dingRobot.py`）。
- **日志**：输出到控制台与 `logs/` 目录。

## 可视化用例生成工具

不想手写 yaml 时，可运行：

```bash
python base/new_testcase_tools.py
```

在弹出的 PyQt5 界面中填写接口信息，先「接口调试」确认接口可用，再「生成 yaml 文件」即可。

## 注意事项

1. `conftest.py`、`pytest.ini` 为固定文件名，不可改名；`pytest.ini` 内容中不要直接加 `#` 中文注释。
2. 除 `data/` 目录下的数据文件可自行增删外，其他文件删除会导致框架无法运行。
3. 三方库版本冲突时，卸载报错的库后按 `requirements.txt` 重新安装即可；国内镜像可用 `https://pypi.tuna.tsinghua.edu.cn/simple/`。
4. `extract.yaml`、`logs/`、`report/`、`venv/`、`__pycache__/`、`.idea/` 均已忽略，不会进入版本库；`conf/config.ini` 中请勿填写真实账号密码。
