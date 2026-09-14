# UO AIGC Model Image Processor

从 Excel `SKU` 和 `修改意见` 列读取服装模特图任务，按 SKU 完成任务筛选、视觉分类、参考图规划、ToAPIs 生成、中间结果管理和日志汇总。

## 基础信息

| 项目 | 内容 |
| --- | --- |
| 名称 | `uo-aigc-model-image-processor` |
| 类型 | Codex Skill 与 Python 批处理脚本 |
| 创建时间 | 2026-07-09（源目录时间） |
| 公开发布整理 | 2026-09-14 |
| 版本 | `2026.09.14` |
| 入口 | [SKILL.md](SKILL.md) |
| 可执行脚本 | [scripts/uo_aigc_batch.py](scripts/uo_aigc_batch.py) |
| 状态 | 已通过本地静态与非付费流程验证 |

## 目标与适用场景

适用于按表格处理服装模特图的缺失角度、动作修改、同款换色、同人替换、平铺图试穿和相关依赖步骤。当 SKU 没有模特图时，可配合 Eagle MCP 选择底图。

不处理单纯的留白、尺寸、分辨率、白底导出或局部裁剪任务。这些项目会被记录为跳过，不会提交给生图接口。

## 输入要求

```text
<ROOT>/
  <任务表>.xlsx
  <SKU-1>/
    <原始模特图、平铺图或细节图>
  <SKU-2>/
    ...
```

- Excel 必须提供 `SKU` 和 `修改意见` 列；每个 SKU 对应根目录的一个直接子目录。
- 支持 JPG、JPEG、PNG 和 WEBP；输入应清晰显示模特身份、服装前后面、版型及需保留细节。
- 引用顺序是位置合同；试穿、换色、背面扩图和同人替换必须按 [SKILL.md](SKILL.md) 的 Figure 规则选图。
- 生成前必须人工核对 `AIGC视觉识别.json` 或 `aigc_image_analysis.json`，不能仅依赖文件名。

## 环境与依赖

- Python 3.12 已用于本地验证；建议使用 Python 3.10 或更新版本。
- 安装 [requirements.txt](requirements.txt) 中的 Pillow、openpyxl 和 requests。
- 生成端使用 ToAPIs；默认模型名称可通过命令行覆盖，有效性以服务商当前支持为准。
- 无模特图流程可选依赖 Eagle 及其 MCP 插件；Windows 下通常从 `%APPDATA%/Eagle/Plugins/mcp-server/` 解析。
- 远程 API 执行生图，本地不要求专用 GPU。

```powershell
python -m pip install -r requirements.txt
```

API 密钥仅通过 `TOAPIS_API_KEY` 环境变量或 `--api-key` 提供。不要将真实密钥、生成日志或带签名的结果链接提交到仓库。本仓库不包含真实密钥。

## 使用方式

将整个目录放入 Codex `skills` 目录，然后通过对话调用，或按三阶段运行：

```powershell
python "<SKILL_DIR>\scripts\uo_aigc_batch.py" --root "<ROOT>" --excel "<TASKS.xlsx>" --mode classify
python "<SKILL_DIR>\scripts\uo_aigc_batch.py" --root "<ROOT>" --excel "<TASKS.xlsx>" --mode plan --image-model "<MODEL_ID>"
$env:TOAPIS_API_KEY = "<YOUR_API_KEY>"
python "<SKILL_DIR>\scripts\uo_aigc_batch.py" --root "<ROOT>" --excel "<TASKS.xlsx>" --mode generate --image-model "<MODEL_ID>" --archive-old
```

`classify` 可在未提供 API 密钥时生成待人工填写的分类记录。`plan` 在不生图的情况下生成任务计划。`generate` 需要 API 密钥并可产生远程费用。

## 交付物

- `AIGC任务筛选.json`：所有修改项的处理或跳过决策。
- `AIGC视觉识别.json` 及 `aigc_image_analysis.json`：图像分类与人工校正数据。
- `AIGC任务计划.json`：按依赖排序的原子生成步骤。
- `AIGC+修改内容+序号`：各 SKU 最终图片。
- `过程文件/`、`AIGC处理日志_<SKU>.txt` 和 `AIGC批处理汇总.json`：中间过程、恢复及审计记录。

默认生图参数是 3:4、2K、600 秒请求超时和最多 3 次尝试。请求前会检查提示词，避免把 SKU、路径、模型名或输出位置发给图像模型。

## 验证范围

发布前应运行 Skill 格式校验、Python 语法检查、CLI 帮助检查，并用临时 Excel/SKU 夹验证 `classify` 和 `plan` 的非付费路径。真实生图需要有效账号、模型与可能产生费用，不包含在本地验证中。
