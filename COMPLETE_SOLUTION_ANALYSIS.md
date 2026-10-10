# AutoSubv3 字幕生成质量问题完整分析报告

## 问题症状回顾

从提供的 SRT 文件可以看到：

```
2
00:00:14,930 --> 00:00:19,160
翻译结果： <font face="Trebuchet MS" size="24"><i>看起来人类似乎要被恶魔们毁灭了。</i></font> [待译]<font face="Trebuchet MS" size="24"><i>但只有一个法师拯救了他们所有人。</i></font>
```

**症状列表**：
1. ❌ 不应该出现的 `翻译结果：` 字符串
2. ❌ HTML 标签 `<font face="Trebuchet MS" size="24">...</font>` 混入字幕内容
3. ❌ 上下文标记 `[待译]` 混入字幕
4. ❌ 字幕内容重复出现（多次处理问题）

---

## 问题根源分析

### 问题 1 & 2：AI 响应后处理不足

**根本原因**：AI 模型在返回翻译时，附加了格式化输出，这些格式被直接纳入字幕内容。

**具体流程**：

```
1. 用户输入字幕:
   "Hello world"

2. 系统发送给 AI:
   "请翻译：Hello world"

3. AI 返回（包含格式）:
   "翻译结果： <font face="Trebuchet MS" size="24"><i>你好世界</i></font>"
   
4. 系统直接采用此响应:
   ❌ item.content = "翻译结果： <font face="Trebuchet MS" size="24"><i>你好世界</i></font>"

5. 正确做法应该是提取纯文本:
   ✅ item.content = "你好世界"
```

**代码位置**：`plugins.v3/autosubv3/translate/openai_translate.py`

**关键问题代码**（第 145-171 行）:

```python
def translate_to_zh(self, text: str, context: str = None, max_retries: int = 3):
    """单条翻译"""
    # ...
    result = completion.choices[0].message.content.strip()
    result = self._clean_text(result)  # ⚠️ 问题：clean_text 只清理SRT格式和空格
    self._last_error = ""
    return True, result  # ❌ 直接返回可能包含 "翻译结果：" 和 HTML 标签的结果
```

**`_clean_text` 方法**（第 107-117 行）分析：

```python
@staticmethod
def _clean_text(text: str) -> str:
    text = text or ""
    text = re.sub(r'\d{2}:\d{2}:\d{2},\d{3}\s*-->\s*\d{2}:\d{2}:\d{2},\d{3}', '', text)  # ✓ 清理时间码
    text = re.sub(r'^\s*\d+\s*$', '', text, flags=re.MULTILINE)  # ✓ 清理序号
    text = re.sub(r'^\s*\d+\s*\n', '', text)
    text = re.sub(r'([\u4e00-\u9fff])\s+([\u4e00-\u9fff])', r'\1\2', text)  # ✓ 清理中文空格
    # ... 其他清理
    return text
    # ❌ 缺少：去除 "翻译结果："
    # ❌ 缺少：去除 HTML 标签
    # ❌ 缺少：去除 [待译] 等上下文标记
```

### 问题 3：上下文泄漏

**位置**：`plugins.v3/autosubv3/pipeline/translation_service.py` 的 `get_context()` 方法

当 `get_context()` 返回的上下文与翻译结果混合时，会出现 `[待译]` 标记：

```python
def get_context(self, all_subs: list, target_indices: List[int], is_batch: bool) -> str:
    context = []
    for idx in range(min_idx, max_idx + 1):
        status = "[待译]" if idx in target_indices else ""  # ⚠️ 这个标记
        content = all_subs[idx].content.replace('\n', ' ').strip()
        context.append(f"{status}{content}")  # 被拼接进上下文
    return "\n".join(context)
```

如果 AI 在某些情况下将这个上下文也输出回来，就会混入字幕。

### 问题 4：重复处理（已在前一份报告中说明）

---

## 完整解决方案

### 方案 A：强化 AI 响应清理（**优先级最高** ⭐⭐⭐⭐⭐）

**文件**：`plugins.v3/autosubv3/translate/openai_translate.py`

**修改 `_clean_text()` 方法**：

```python
@staticmethod
def _clean_text(text: str) -> str:
    """彻底清理 AI 响应中的垃圾内容"""
    text = text or ""
    
    # 1. 移除常见的前缀标记
    text = re.sub(r'^.*?(?:翻译结果|翻译如下|译文|结果)[:：\s]*', '', text, flags=re.MULTILINE)
    
    # 2. 移除 HTML 标签（包括 font, span, div 等）
    text = re.sub(r'<[^>]+>', '', text, flags=re.MULTILINE)
    
    # 3. 移除 HTML 实体编码
    text = text.replace('&nbsp;', ' ').replace('&lt;', '<').replace('&gt;', '>')
    text = text.replace('&amp;', '&').replace('&quot;', '"').replace('&#39;', "'")
    
    # 4. 移除上下文标记
    text = re.sub(r'\s*\[待译\]\s*', '', text)
    text = re.sub(r'\s*\[翻译失败\]\s*', '', text)
    
    # 5. 移除 SRT 格式残留
    text = re.sub(r'\d{2}:\d{2}:\d{2},\d{3}\s*-->\s*\d{2}:\d{2}:\d{2},\d{3}', '', text)
    text = re.sub(r'^\s*\d+\s*$', '', text, flags=re.MULTILINE)
    text = re.sub(r'^\s*\d+\s*\n', '', text)
    
    # 6. 修复中文/英文间距
    text = re.sub(r'([\u4e00-\u9fff])\s+([\u4e00-\u9fff])', r'\1\2', text)
    text = re.sub(r'([\u4e00-\u9fff])\s+([a-zA-Z0-9])', r'\1\2', text)
    text = re.sub(r'([a-zA-Z0-9])\s+([\u4e00-\u9fff])', r'\1\2', text)
    
    # 7. 清理多余空白
    text = re.sub(r'\s+\n', '\n', text)
    text = re.sub(r'\n\s+', '\n', text)
    text = re.sub(r'\s+', ' ', text).strip()
    
    # 8. 最后确保每行都被清理过
    lines = [line.strip() for line in text.split('\n')]
    return '\n'.join(line for line in lines if line)
```

**改进 `translate_to_zh()` 方法**（第 168-170 行）:

```python
def translate_to_zh(self, text: str, context: str = None, max_retries: int = 3):
    """单条翻译：走 3.5.10 prompt 思路"""
    text = self._clean_text(text)
    context = self._clean_text(context) if context else None

    system_prompt = """您是一位专业字幕翻译专家，请严格遵循以下规则：
1. 将原文精准翻译为简体中文，保持原文本意
2. 使用自然的口语化表达，符合中文观影习惯
3. 结合上下文语境，人物称谓、专业术语、情感语气在上下文中保持连贯
4. 按行翻译待译内容。翻译结果不要包括上下文。
5. 输出内容必须仅包括译文。不要输出任何开场白，解释说明或总结。
6. 不要输出任何HTML标签、格式标记、或前缀文字。
7. 遇到英文脏话时请翻译成自然中文口语，不要保留英文单词"""
    
    user_prompt = f"翻译上下文：\n{context}\n\n需要翻译的内容：\n{text}" if context else f"请翻译：\n{text}"

    last_error = ""
    for attempt in range(max_retries + 1):
        try:
            completion = self.__get_model(
                message=user_prompt,
                system_hint=system_prompt,
                temperature=0.2,
                top_p=0.9
            )
            result = completion.choices[0].message.content.strip()
            result = self._clean_text(result)  # ⭐ 关键：再次清理
            
            # ⭐ 新增：二次验证，确保结果干净
            if not result or len(result) > 500 or any(tag in result.lower() for tag in ['<', '>', 'html', '翻译结果']):
                # 响应可能不清洁，强制再次清理
                result = re.sub(r'<[^>]+>', '', result)  # 移除任何剩余HTML
                result = re.sub(r'^[^：：]*[：：]', '', result)  # 移除前缀
                result = result.strip()
            
            self._last_error = ""
            return True, result
        except Exception as e:
            # ... 错误处理保持不变 ...
```

**改进 `translate_batch_to_zh()` 方法**（第 304-306 行）:

```python
# 构建结果
translations: List[Optional[str]] = [None] * len(texts)
for item in output_batch:
    idx = int(item["id"]) - 1
    zh = item.get("zh", "").strip()  # 获取原始响应
    
    # ⭐ 新增：清理 zh 字段
    zh = self._clean_text(zh)  # 二次清理
    zh = re.sub(r'<[^>]+>', '', zh)  # 移除 HTML 标签
    zh = re.sub(r'^[^：：]*[：：]\s*', '', zh)  # 移除前缀
    
    if 0 <= idx < len(translations):
        translations[idx] = zh
```

---

### 方案 B：改进 AI Prompt（预防性措施）

**修改 System Prompt**（第 150-156 行），更严格地约束输出格式：

```python
system_prompt = """您是一位专业字幕翻译专家，必须遵循以下规则：

==== 翻译规则 ====
1. 精准翻译为简体中文，���持原文本意
2. 自然的口语化表达，符合中文观影习惯
3. 结合上下文语境，保持连贯性
4. 按行翻译待译内容，不包括上下文

==== 输出格式要求 ====
⚠️ 严格要求：
- 仅输出纯文本翻译内容，每行一个译文
- 禁止输出任何前缀文字（如"翻译结果："、"译文："等）
- 禁止输出任何HTML标签（<tag>、</tag>等）
- 禁止输出任何标记或符号（[、]、{、}等）
- 禁止输出任何说明文字或总结
- 禁止输出代码块标记（```、~~~等）

==== 示例 ====
输入："Hello, world!"
输出："你好，世界！"

输入："Good morning."
输出："早上好。"

==== 重要提示 ====
如果您无法确定翻译内容是否包含非文本元素（如HTML、前缀等），
请再次检查并确保输出是100%的纯文本。
"""
```

---

### 方案 C：后处理验证（保险措施）

**在 `translation_service.py` 中添加验证函数**：

```python
class TranslationService:
    # ... 现有代码 ...
    
    @staticmethod
    def _validate_translation_output(text: str) -> str:
        """验证翻译输出的合法性"""
        if not text:
            return text
        
        # 检查禁止内容列表
        forbidden_patterns = [
            r'<[^>]+>',  # HTML 标签
            r'翻译结果|翻译如下|译文|结果',  # 前缀文字
            r'\[待译\]|\[翻译|翻译失败\]',  # 上下文标记
            r'```|~~~|```',  # 代码块标记
            r'^[\s]*【|】',  # 中文括号标记
        ]
        
        for pattern in forbidden_patterns:
            matches = re.findall(pattern, text, re.MULTILINE)
            if matches:
                # 发现禁止内容，清理它们
                text = re.sub(pattern, '', text, flags=re.MULTILINE)
        
        return text.strip()
    
    def format_translated_content(self, original: Any, translated: Any, output_mode: str = None) -> str:
        trans = self.normalize_subtitle_text_line(translated)
        origin = self.normalize_subtitle_text_line(original)
        
        # ⭐ 新增：验证翻译输出
        trans = self._validate_translation_output(trans)
        
        if (output_mode or self._output_mode()) == 'chinese_only':
            return trans
        if not origin:
            return trans
        if not trans:
            return origin
        
        # ⭐ 检查重复处理
        if trans and origin.startswith(trans):
            return origin
        
        return f"{trans}\n{origin}"
```

**在 `process_single()` 中调用验证**（第 137-154 行）:

```python
def process_single(self, all_subs: List[srt.Subtitle], item: srt.Subtitle, translate_to_zh=None) -> srt.Subtitle:
    idx = all_subs.index(item)
    context = self.get_context(all_subs, [idx], is_batch=False) if self._context_window() > 0 else None
    success, trans = (translate_to_zh or self.translate_to_zh)(item.content, context)

    if success:
        # ⭐ 新增：验证翻译结果
        trans = self._validate_translation_output(trans)
        
        item.content = self.format_translated_content(item.content, trans)
        self._stats()['line_fallback'] += 1
        self._stats()['translated'] += 1
        return item

    self._remember_api_error(trans)
    if self._output_mode() == 'chinese_only':
        item.content = "[翻译失败]"
    else:
        item.content = self.format_translated_content(item.content, "[翻译失败]")
    self._stats()['failed'] += 1
    return item
```

---

## 实施顺序（按优先级）

### 第一步（立即实施）⭐⭐⭐⭐⭐

在 `openai_translate.py` 中强化 `_clean_text()` 方法，增加清理规则。

```bash
# 修改文件
plugins.v3/autosubv3/translate/openai_translate.py
  - 扩展 _clean_text() 方法
  - 优化 translate_to_zh() 的结果验证
  - 优化 translate_batch_to_zh() 的结果处理
```

### 第二步（本周内）⭐⭐⭐⭐

改进 AI Prompt，使模型输出更加规范。

```bash
# 修改文件
plugins.v3/autosubv3/translate/openai_translate.py
  - 增强 system_prompt
  - 添加明确的输出格式要求
```

### 第三步（本周末）⭐⭐⭐

在翻译服务中添加后处理验证。

```bash
# 修改文件
plugins.v3/autosubv3/pipeline/translation_service.py
  - 添加 _validate_translation_output() 方法
  - 在 format_translated_content() 中调用验证
  - 在 process_single() 中调用验证
```

### 第四步（可选加固）⭐⭐

修复重复处理问题（参考前一份报告）。

```bash
# 修改文件
plugins.v3/autosubv3/pipeline/translation_service.py
  - 保存原始内容副本
  - 在降级时恢复原始内容
```

---

## 验证方法

### 测试用例 1：检查 HTML 标签清理

```python
# 输入
input_str = "翻译结果： <font face='Trebuchet MS' size='24'><i>你好世界</i></font>"

# 预期输出
expected = "你好世界"

# 验证
assert OpenAi._clean_text(input_str) == expected
```

### 测试用例 2：检查前缀移除

```python
inputs = [
    "翻译如下：你好世界",
    "译文：你好世界",
    "结果：你好世界",
    "翻译：你好世界"
]

for inp in inputs:
    assert OpenAi._clean_text(inp) == "你好世界"
```

### 测试用例 3：检查上下文标记清理

```python
input_str = "[待译] 你好世界 [翻译失败]"
expected = "你好世界"
assert OpenAi._clean_text(input_str) == expected
```

### 测试用例 4：实际生成测试

运行字幕生成，检查输出 SRT：

```bash
# 应该看到纯净的翻译
1
00:00:00,000 --> 00:00:05,000
你好世界
原始英文

# 而不是
1
00:00:00,000 --> 00:00:05,000
翻译结果： <font>你好世界</font>
[待译] 原始英文
```

---

## 总结

| 问题 | 根源 | 解决方案 |
|------|------|---------|
| `翻译结果：` 出现 | AI 响应包含前缀 | `_clean_text()` 中添加移除规则 |
| `<font>` 标签混入 | AI 响应包含 HTML | `_clean_text()` 中添加去标签正则 |
| `[待译]` 混入 | 上下文泄漏或被包含 | `_clean_text()` 中移除上下文标记 |
| 字幕重复 | 多次调用 `format_translated_content()` | 检测已处理格式，避免重复处理 |

所有问题都**可以通过强化 `_clean_text()` 方法完全解决**。
