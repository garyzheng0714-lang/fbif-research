# fbif-research

![类型](https://img.shields.io/badge/%E7%B1%BB%E5%9E%8B-FBIF%20Skill%20%E5%B7%A5%E5%85%B7-2563eb)
![技术栈](https://img.shields.io/badge/%E6%8A%80%E6%9C%AF%E6%A0%88-Python%20%7C%20Bash%20%7C%20Feishu-0f766e)
![状态](https://img.shields.io/badge/%E7%8A%B6%E6%80%81-%E5%8F%AF%E6%89%A7%E8%A1%8C%E8%84%9A%E6%89%8B%E6%9E%B6-16a34a)
![README](https://img.shields.io/badge/README-%E4%B8%AD%E6%96%87-111827)

FBIF 食品饮料品牌深度调研 skill 包，把来源搜索、全文抓取、逐段翻译、质量检查、HTML 组装、OSS 部署和飞书多维表格写回拆成可执行流程。

## 仓库定位

| 项目 | 说明 |
| --- | --- |
| 分类 | FBIF 工具 / Claude-Codex skill / 品牌研究脚手架 |
| 面向对象 | 需要批量产出食品饮料品牌深度调研材料、保留原文与中文翻译证据链的研究流程 |
| 主要产物 | 每个品牌的 `manifest.json`、来源清单、原文 Markdown、翻译内容和最终 HTML 报告 |
| 边界 | 本仓库提供流程、脚本、模板和质量门槛；具体调研内容由每次执行时的来源搜索和抓取结果决定 |

## 功能概览

- 初始化品牌研究产物目录，并复制锁定版本的脚本和模板。
- 维护 URL 来源清单和分模块质量门槛。
- 通过 x-reader 或 Jina Reader fallback 抓取来源全文。
- 保存原文 Markdown，并用脚本插入中文翻译。
- 分阶段质量检查：来源覆盖、抓取完整性、翻译质量和最终 HTML。
- 组装单页 HTML 报告，并支持阿里云 OSS 上传。
- 通过飞书多维表格读取待调研品牌、写回完成状态。
- `research-loop.sh` 支持批量主循环、锁文件、硬超时、dry-run、limit 和每品牌日志。

## 快速开始

安装基础依赖：

```bash
pip install oss2 requests
```

可选安装更强的来源抽取工具：

```bash
pip install -e /path/to/x-reader
```

初始化一个品牌调研产物目录：

```bash
python3 scripts/init.py "Taifun" "Taifun" \
  --company "Taifun-Tofu GmbH" \
  --country "Germany" \
  --category "Organic Tofu" \
  --output-dir outputs/taifun
```

初始化后会创建：

```text
outputs/<brand>/
├── manifest.json
├── source-inventory.json
├── sources/
├── scripts/
├── templates/
└── final/
```

## 标准流程

1. 搜索并把来源 URL 登记到 `source-inventory.json`。
2. 检查来源覆盖：

   ```bash
   python3 scripts/check_quality.py <artifact_root> --step 1
   ```

3. 批量抓取来源：

   ```bash
   python3 scripts/fetch_source.py <artifact_root> --batch
   ```

4. 使用 `scripts/add_translation.py` 插入中文翻译。
5. 检查翻译质量：

   ```bash
   python3 scripts/check_quality.py <artifact_root> --step 3
   ```

6. 清洗、组装、部署：

   ```bash
   python3 scripts/clean_content.py <artifact_root>
   python3 scripts/assemble_single.py <artifact_root>
   python3 scripts/deploy_oss.py <artifact_root>
   ```

7. 审查最终报告：

   ```bash
   python3 scripts/audit_report.py <artifact_root>
   ```

## 批量调研

`research-loop.sh` 会从飞书多维表格读取下一个待调研品牌，启动隔离会话，验证产物，并在成功后写回完成状态。

```bash
bash research-loop.sh
bash research-loop.sh --limit 10
bash research-loop.sh --dry-run --limit 3
```

脚本内置：

- 每品牌独立锁文件，避免重复执行。
- 90 分钟硬超时和 `--max-turns` 控制。
- 启动时清理超过 3 小时的残留锁。
- 完成验证门控，不通过则不写回。
- 每品牌独立日志。

## 项目结构

```text
.
├── SKILL.md                    # skill 入口说明和执行规则
├── research-loop.sh            # 批量品牌调研主循环
├── references/                 # 检索方法、模块指南、审查提示和 HTML 规范
├── scripts/
│   ├── init.py                 # 创建品牌研究产物目录
│   ├── fetch_source.py         # 抓取单个 URL 或批量抓取 source-inventory.json
│   ├── save_source.py          # 保存抓取后的来源 Markdown
│   ├── add_translation.py      # 向来源文件插入中文翻译
│   ├── clean_content.py        # 清理噪音内容
│   ├── check_quality.py        # 分阶段质量检查
│   ├── assemble_single.py      # 组装单页 HTML
│   ├── audit_report.py         # 审查 HTML 报告质量
│   ├── deploy_oss.py           # 上传最终报告到 OSS
│   ├── bitable_read.py         # 从飞书多维表格读取待调研品牌
│   ├── bitable_write.py        # 写回完成数据
│   └── validate_completion.py  # 批量主循环的完成验证
└── templates/
    ├── single-page-template.html
    └── appendix-template.html
```

## 配置

常见配置来源：

- `bitable-config.json`：飞书 app/table 配置。
- `FEISHU_APP_ID` 和 `FEISHU_APP_SECRET`：没有配置文件时的飞书凭证来源。
- `oss-config.json`：产物目录内的 OSS 部署配置。

请把所有凭证留在本地环境或私有配置文件中，不要提交到版本库。

## 备注

- 搜索、抓取、翻译、组装、审查是独立阶段，适合断点续跑。
- `scripts/` 和 `templates/` 会复制到每个产物目录，保证单次调研使用固定版本工具。
- 最终 HTML 默认位于 `<artifact_root>/final/report.html`。
