# Vision API SOP

## ⚠️ 前置规则（必须遵守）

1. **先枚举窗口**：调用 vision 前必须先用 `pygetwindow` 枚举窗口标题，确认目标窗口存在且已激活到前台。窗口不存在就不要截图。
2. **🚫 禁止全屏截图**：必须先利用ljqCtrl截取窗口区域。能截局部（如标题栏）就不截整窗口，能截窗口就绝不全屏。全屏截图在任何场景下都不允许。
3. **能不用 vision 就不用**：如果窗口标题/本地 OCR（`ocr_utils.py`）能获取所需信息，就不要调用 vision API，省 token 且更可靠。Vision 是最后手段。

## 快速用法

```python
from vision_api import ask_vision
result = ask_vision(image, prompt="描述图片内容", backend="claude", timeout=60, max_pixels=1_440_000)
# image: 文件路径(str/Path) 或 PIL Image
# backend: 'claude'(默认) | 'openai' | 'modelscope'
# 返回 str：成功为模型回复，失败为 'Error: ...'
```

## 如果没有 `vision_api.py`，初次构建vision能力

1. 复制 `memory/vision_api.template.py` → `memory/vision_api.py`
2. 只改头部"用户配置区"：去 `mykey.py` 里扫描变量名（⚠️ 只看名字，禁止输出 apikey 值），尝试找能用配置名填入 `CLAUDE_CONFIG_KEY` / `OPENAI_CONFIG_KEY`，`DEFAULT_BACKEND` 选后端，并测试
3. 保底：没有可用 config 时去 `https://modelscope.cn/my/myaccesstoken` 申请 token 填入 `MODELSCOPE_API_KEY`

## 📦 大批量图片识别规范（>5张，强制使用）

### 核心策略：主agent直接循环 + 及时保存进度

> 模型已升级至 Qwen3-VL-8B，单张识别约10秒，60张以内通常不超600秒。

### 执行流程

```
1. 收集图片路径列表
2. 每批5张，逐张串联识别
3. 每识别完一张：
   - 立即追加写入 vision识别结果.json
   - 输出进度（如 "3/30 image_xxx.png: 识别成功"）
4. 每批完成后输出批次摘要
5. 全部完成后返回汇总统计
```

### 进度文件格式（vision识别结果.json）

```json
[
  {
    "filename": "image1.png",
    "time": "2024-12-01 10:30:00",
    "raw_text": "识别的完整原始文本内容..."
  },
  {
    "filename": "image2.png",
    "time": "2024-12-01 10:30:15",
    "raw_text": "Error: timeout"
  }
]
```

### 超时与重试配置

| 配置项 | 值 | 说明 |
|--------|------|------|
| 单次 Vision 请求 | `timeout=120` | 与 vision_api 默认一致 |
| 失败重试 | 自动重试1次 | 网络波动/排队导致 |
| 批次大小 | 5张/批 | 平衡进度反馈与效率 |

### 异常处理

| 异常 | 处理方式 |
|------|----------|
| 单次识别超时 | 重试1次，仍失败则标记"失败"继续下一张 |
| 接近600秒超时 | 保存当前进度，告知用户剩余数量，可选择继续 |
| 需要中断 | 直接停止，进度已保存可随时恢复 |
