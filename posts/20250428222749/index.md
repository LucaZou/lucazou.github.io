# Coze如何解除代码块中的依赖库限制？

&gt;

&lt;!--more--&gt;
# 案例分析：基于Pandas的数据处理插件开发

在使用Coze工作流（Workflow）编排自动化任务时，我们经常会用到「代码块」(本文针对Python语言)来执行自定义逻辑。但如果你深入开发过，就会发现一个**隐秘的限制**：

&gt; **Coze工作流内置的Python代码块，对第三方库支持非常有限**，尤其是像 `pandas`、`openpyxl` 这样的重量级依赖，经常无法直接使用，在 Python 环境中，仅内置了两个第三方依赖库：requests_async 和 numpy。[官方使用说明](https://www.coze.cn/open/docs/guides/code_node)

那有没有办法**突破这一限制**，在Coze里灵活使用各种Python库，执行更复杂的逻辑呢？  
答案是——**开发自定义插件（Plugin）！**

这篇文章将通过一个实战案例，教你如何绕过代码块的依赖限制，自由调用Pandas来处理Excel数据。

---

## 问题背景：Coze代码块的局限

Coze官方文档中提到，代码块运行环境默认只支持部分基础Python库，如：

![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202504282220701.png)

而像 `pandas`、`requests` 这类第三方库，往往是**不支持的**。即使你在代码中导入了它们，也会报错：

```bash
ModuleNotFoundError: No module named &#39;pandas&#39;
```

这使得很多数据处理、文件解析、网络请求等需求，难以直接用原生代码块完成。

---

## 解决方案：开发插件（Plugin）

Coze允许开发者**自定义插件**，通过独立服务或Coze托管运行插件代码。这种方式拥有**更完整的Python环境支持**，可以自由安装依赖包，极大扩展了功能边界。

比如，我开发了一个叫 `Exec_Code` 的插件，它支持：

- 在线下载指定的Excel文件
- 执行任意Pandas数据处理代码
- 返回处理后的结果，供后续流程继续使用

### 插件功能概览

- **输入参数**：
    
    - `files`: Excel文件的URL地址
    - `code_block`: 要执行的Pandas代码片段
- **使用依赖库**：
    
    - `pandas`：数据处理
    - `openpyxl`：解析Excel
    - `pydantic`：参数校验
    - `requests`：文件下载
- **核心逻辑**：
    
    1. 下载并读取Excel文件（支持多Sheet）
    2. 安全执行用户提供的代码块
    3. 强制要求代码块输出 `result` 变量
    4. 自动将DataFrame或复杂对象转换为JSON返回

---

## 插件核心代码解析

这里简单拆解一下主要逻辑：

### 1. 下载并读取Excel文件

```python
response = requests.get(str(input_args.files), timeout=10)
sheets_dict = pd.read_excel(BytesIO(response.content), sheet_name=None)
```

通过`requests`获取文件，通过`pandas`一次性读取**全部Sheet**成字典 `{sheet_name: DataFrame}`。

---

### 2. 安全执行Pandas代码块

为了避免执行恶意代码，定义了一个**受限版的globals**环境，只允许部分内置函数和必要模块（pandas、json）：

```python
safe_globals = {
    &#39;__builtins__&#39;: {
        &#39;print&#39;: print,
        &#39;len&#39;: len,
        ...
    },
    &#39;pd&#39;: pd,
    &#39;json&#39;: json,
}
local_vars = {
    &#39;files&#39;: sheets_dict, 
    &#39;result&#39;: None
}
exec(input_args.code_block, safe_globals, local_vars)
```

这样，即使是用户自定义的代码，也可以保证一定程度的安全性。

---

### 3. 强制要求输出`result`

所有用户的Pandas逻辑，**必须赋值给`result`** 变量，否则插件会直接返回错误提示。

```python
if &#39;result&#39; not in local_vars:
    return PluginOutput(status=&#34;error&#34;, error=&#34;代码块未定义 result 变量&#34;)
```

这样可以标准化输出，保证后续节点能正确使用结果。

---

### 4. 结果处理与序列化

Pandas的DataFrame、复杂对象，默认是无法直接转成JSON的。因此，写了一个递归函数 `convert_to_serializable`，确保输出最终可JSON化。

```python
if isinstance(result, pd.DataFrame):
    result = {&#39;json_result&#39;: json.dumps(result.to_dict(orient=&#39;records&#39;), ensure_ascii=False)}
else:
    result = convert_to_serializable(result)
```

---

## 使用效果示例

假设我有一个Excel，里面是销售数据，想要提取某个Sheet的总销售额。

可以在Coze工作流中这样调用插件：

- `files`: &#34;[https://example.com/sales.xlsx](https://example.com/sales.xlsx)&#34;
- `code_block`:

```python
df = files[&#34;Sheet1&#34;]
total_sales = df[&#34;销售额&#34;].sum()
result = {&#34;total_sales&#34;: total_sales}
```

执行后，插件返回：

```json
{
  &#34;status&#34;: &#34;success&#34;,
  &#34;result&#34;: {
    &#34;total_sales&#34;: 128000
  }
}
```

是不是比在代码块中苦苦挣扎安装库、处理异常方便多了？

---

## 小结

总结一下：

|问题|解决方案|
|:--|:--|
|Coze代码块无法直接用第三方库|开发插件，运行在自由环境|
|想用Pandas、Requests等|在插件中声明依赖|
|保证安全执行动态代码|使用受限globals &#43; 控制输出变量|

插件的方式，不仅解锁了更多强大的Python能力，还能让你的Coze工作流更加灵活、强大！

**插件商店搜索 ’Exec_Code&#39; 立即体验！**
![image.png](https://img-1313049298.cos.ap-shanghai.myqcloud.com/note-img/202504282227214.png)

---

## 附：完整插件源码

```python
from runtime import Args
from typings.exec2.exec2 import Input, Output
from pydantic import BaseModel, HttpUrl
import pandas as pd
import requests
from io import BytesIO
from typing import Optional, Dict
import logging
import json

# 输入模型
class PluginInput(BaseModel):
    files: HttpUrl  # 单一 Excel 文件的 URL
    code_block: str  # 要执行的 Pandas 代码块

# 输出模型
class PluginOutput(BaseModel):
    status: str  # 执行状态（success 或 error）
    result: Optional[dict] = None  # 执行结果，需为字典格式
    error: Optional[str] = None  # 错误信息（如果有）

def convert_to_serializable(obj):
    &#34;&#34;&#34;
    递归地将对象转换为 JSON 可序列化格式，处理 DataFrame 和多 sheet 字典。
    
    Args:
        obj: 要转换的对象（例如 DataFrame、字典、列表等）
        
    Returns:
        JSON 可序列化对象（dict、list、str、int、float、bool、None）
    &#34;&#34;&#34;
    logger = logging.getLogger(__name__)
    if isinstance(obj, pd.DataFrame):
        logger.info(f&#34;Converting DataFrame to JSON string: {obj.shape}&#34;)
        data = obj.to_dict(orient=&#39;records&#39;)
        return json.dumps(data, ensure_ascii=False)  # 确保中文字符正确编码
    elif isinstance(obj, dict):
        # 处理多 sheet 字典或其他字典类型
        return {key: convert_to_serializable(value) for key, value in obj.items()}
    elif isinstance(obj, list):
        return [convert_to_serializable(item) for item in obj]
    elif isinstance(obj, (int, float, str, bool)) or obj is None:
        return obj
    else:
        logger.warning(f&#34;Unsupported type {type(obj)} in result, converting to str&#34;)
        return str(obj)

def handler(args: Args[Input]) -&gt; Output:
    &#34;&#34;&#34;
    执行 Pandas 代码块，支持加载 Excel 文件的所有 sheet。
    
    Parameters:
    - args.input.files: 单一 Excel 文件的 URL
    - args.input.code_block: 要执行的 Pandas 代码（必须赋值给 &#39;result&#39;）
    - args.logger: 日志记录器实例
    
    Returns:
    - PluginOutput 包含状态、结果和错误信息
    &#34;&#34;&#34;
    logger = args.logger or logging.getLogger(__name__)
    logger.info(&#34;Starting PandasCodeExecutor plugin&#34;)

    try:
        # 验证输入
        input_args = PluginInput(
            files=args.input.files,
            code_block=args.input.code_block
        )
        logger.info(&#34;Input validation passed&#34;)

        # 检查输入是否为空
        if not input_args.files:
            logger.error(&#34;No file URL provided&#34;)
            return PluginOutput(status=&#34;error&#34;, error=&#34;未提供文件 URL&#34;)
        if not input_args.code_block.strip():
            logger.error(&#34;Code block is empty&#34;)
            return PluginOutput(status=&#34;error&#34;, error=&#34;代码块为空&#34;)

        # 下载并读取 Excel 文件的所有 sheet
        try:
            response = requests.get(str(input_args.files), timeout=10)
            response.raise_for_status()
            # 使用 sheet_name=None 加载所有 sheet，返回 {sheet_name: DataFrame} 字典
            sheets_dict = pd.read_excel(BytesIO(response.content), sheet_name=None)
            logger.info(f&#34;Successfully loaded {len(sheets_dict)} sheets from {input_args.files}: {list(sheets_dict.keys())}&#34;)
        except requests.RequestException as e:
            logger.error(f&#34;Failed to download file {input_args.files}: {str(e)}&#34;)
            return PluginOutput(status=&#34;error&#34;, error=f&#34;下载文件失败: {str(e)}&#34;)
        except pd.errors.ParserError as e:
            logger.error(f&#34;Failed to parse Excel file {input_args.files}: {str(e)}&#34;)
            return PluginOutput(status=&#34;error&#34;, error=f&#34;解析Excel文件失败: {str(e)}&#34;)

        # 设置安全的执行环境
        safe_globals = {
            &#39;__builtins__&#39;: {
                &#39;print&#39;: print,
                &#39;len&#39;: len,
                &#39;range&#39;: range,
                &#39;str&#39;: str,
                &#39;int&#39;: int,
                &#39;float&#39;: float,
                &#39;list&#39;: list,
                &#39;dict&#39;: dict,
                &#39;set&#39;: set,
                &#39;tuple&#39;: tuple,
                &#39;bool&#39;: bool,
                &#39;abs&#39;: abs,
                &#39;max&#39;: max,
                &#39;min&#39;: min,
                &#39;sum&#39;: sum,
                &#39;round&#39;: round,
                &#39;zip&#39;: zip,
            },
            &#39;pd&#39;: pd,  # 提供 pandas 库
            &#39;json&#39;: json  # 提供 json 库
        }

        # 准备本地变量
        local_vars = {
            &#39;files&#39;: sheets_dict,  # 提供所有 sheet 的字典 {sheet_name: DataFrame}
            &#39;result&#39;: None  # 用于存储 code_block 的输出
        }

        # 执行代码块
        try:
            exec(input_args.code_block, safe_globals, local_vars)
            logger.info(&#34;Code block executed successfully&#34;)
        except Exception as e:
            logger.error(f&#34;Code execution failed: {str(e)}&#34;)
            logger.info(f&#34;Code block: {input_args.code_block}&#34;)
            return PluginOutput(status=&#34;error&#34;, error=f&#34;代码执行失败: {str(e)}&#34;)

        # 获取结果
        if &#39;result&#39; not in local_vars:
            logger.error(&#34;Code block did not define &#39;result&#39; variable&#34;)
            return PluginOutput(status=&#34;error&#34;, error=&#34;代码块未定义 result 变量&#34;)

        result = local_vars.get(&#39;result&#39;)
        logger.info(f&#34;Result type: {type(result)}, value: {result}&#34;)
        if result is None:
            logger.warning(&#34;No result assigned in code block&#34;)
            return PluginOutput(status=&#34;success&#34;, error=&#34;代码块未赋值给result&#34;)

        # 转换为 JSON 可序列化格式
        if isinstance(result, pd.DataFrame):
            result = {&#39;json_result&#39;: json.dumps(result.to_dict(orient=&#39;records&#39;), ensure_ascii=False)}
        else:
            result = convert_to_serializable(result)
        logger.info(f&#34;Converted result: {result}&#34;)
        logger.info(&#34;Result processed for output&#34;)

        # 验证 result 是否为字典
        if not isinstance(result, dict):
            logger.error(f&#34;Result is not a dictionary: {type(result)}&#34;)
            return PluginOutput(status=&#34;error&#34;, error=f&#34;result 必须是字典，实际类型为 {type(result)}&#34;)

        try:
            return PluginOutput(status=&#34;success&#34;, result=result)
        except Exception as e:
            logger.error(f&#34;Failed to create PluginOutput: {str(e)}&#34;)
            return PluginOutput(status=&#34;error&#34;, error=f&#34;无法序列化 result: {str(e)}&#34;)

    except Exception as e:
        logger.error(f&#34;Plugin execution failed: {str(e)}&#34;)
        return PluginOutput(status=&#34;error&#34;, error=str(e))
```


---

> 作者: [LucaZou](https://github.com/LucaZou)  
> URL: https://lucazou.github.io/posts/20250428222749/  

