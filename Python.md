# Python

### Pytest(test in python)

**1. 安装 pytest**

首先，安装测试框架：

```
pip install pytest
```

------

**2. 编写测试类与方法**

在 Python 中，你不需要像 Spring 那样通过注解（如 `@Test`）来标记测试，而是通过**命名约定**：

- 测试文件名必须以 `test_` 开头（或以 `_test` 结尾）。
- 测试类名必须以 `Test` 开头（不需要继承任何类）。
- 测试方法名必须以 `test_` 开头。

**3. 使用 Fixtures（类似于 Spring 的 @Before / @Bean）**

Spring 开发者最喜欢的依赖注入，在 pytest 中通过 `fixture` 实现。它可以用来初始化数据库连接、加载配置或 Mock 对象。

```
import pytest

@pytest.fixture
def sample_user():
    # 相当于设置测试数据
    return {"id": 1, "name": "Alice"}

def test_user_name(sample_user):
    # 直接将 fixture 名称作为参数传入，pytest 会自动注入
    assert sample_user["name"] == "Alice"
```

------

**4. 命令行执行pytest**

在项目根目录下运行以下命令(.venv的上级目录)，pytest 会自动扫描所有符合命名规则的文件并执行：

```
pytest          # 执行所有测试
pytest -v       # 详细输出 (Verbose)
pytest test_file.py::TestClass::test_method  # 只运行特定测试
```



```python
python -m pytest tests/test_chunker.py -v -s # 这里-m是强行将当前执行命令的目录加入到系统的环境变量 `sys.path` 中，以便pytest执行。-s 显示代码中的 `print()` 语句。
```

### 语法糖

#### **列表推导式 (List Comprehension)**

```python
#strip()去空格 python默认""为false 
#如果p去空格 还有内容则保留，最终得到一个列表
paragraphs = [p.strip() for p in text.split("\n") if p.strip()]

texts = [c["chunk_text"] for c in chunks]
```

一般写法：**[x for x in list if cond]**

#### enumerate()

`enumerate` 的字面意思是“枚举”。它接收一个可迭代对象（如 List），并将其转换成一个由 `(索引, 元素)` 组成的元组（Tuple）序列。

```
for idx, chunk_text in enumerate(page_chunks):
```

page_chunks: 是上一步 split_text 返回的字符串列表 ["块1", "块2", "块3"]。

enumerate(page_chunks): 会生成类似 (0, "块1"), (1, "块2"), (2, "块3") 的数据流。

idx, chunk_text: 这是 变量解构 (Destructuring)。它直接把元组里的第一个值（索引）赋给 idx，第二个值（内容）赋给 chunk_text。

#### `zip(chunks, vectors)`：拉链操作

在 Java 中，如果你有两个长度相同的 List 要同时遍历，你通常得通过 `list1.get(i)` 和 `list2.get(i)`。

- **Python 的做法：** `zip` 像拉链一样，把两个列表对应的元素一对一“扣”在一起，每次循环直接给你一个包含两个元素的元组 `(chunk, vec)`。

example:

```
for i, (chunk, vec) in enumerate(zip(chunks, vectors)):
```

### .env

在 Python 开发中，使用 `.env` 文件是管理**数据库密码、API 密钥**等敏感信息的标准做法。它可以确保这些私密数据不会被意外上传到 GitHub。

以下是配置和使用的三步法：

------

**1. 安装库**

最常用的库是 `python-dotenv`。打开终端运行：

```
pip install python-dotenv
```

------

**2. 创建 `.env` 文件**

在项目的**根目录**（与你的 `.py` 主文件同级）创建一个名为 `.env` 的文件（注意前面有一个点）。

写入内容示例：

```python
# 不要加引号，除非值中包含空格
API_KEY=your_secret_key_123
DATABASE_URL=mysql://user:pass@localhost/db
DEBUG=True
```

> **注意：** 记得将 `.env` 添加到你的 `.gitignore` 文件中，防止它被提交到仓库。

------

**3. 在 Python 中读取**

在代码中使用 `load_dotenv()` 函数将变量加载到系统的环境变量中，然后通过 `os.getenv` 读取。

```python
import os
from dotenv import load_dotenv

# 1. 加载 .env 文件内容
load_dotenv()

# 2. 获取环境变量
api_key = os.getenv("API_KEY")
db_url = os.getenv("DATABASE_URL")

# 3. 使用变量
print(f"正在连接数据库: {db_url}")
print(f"API 密钥长度: {len(api_key)}")
```

------

**💡 进阶小技巧**

- 默认值： 如果 `.env` 里没写某个变量，为了防止程序崩溃，可以设置默认值： `port = os.getenv("PORT", "8080")`
- 强制加载： 如果你想确保系统原有的同名变量被 `.env` 覆盖，可以使用： `load_dotenv(override=True)`
- 生产环境： 在 Linux 服务器或 Docker 上部署时，通常直接在系统中设置环境变量，代码逻辑依然保持 `os.getenv` 即可，无需更改。

   **pydantic加持：**

可以使用用 **Pydantic** 自动读取 `.env` 配置，并生成一个强类型配置对象 `settings`，该对象的字段与`.env` 配置字段一一对应，可直接在项目代码中引用