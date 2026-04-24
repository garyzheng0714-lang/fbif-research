# FBIF Research

## Overview

FBIF Research 是一套食品饮料品牌深度调研脚手架和执行工具。它面向品牌研究流程，把“搜索来源、抓取全文、逐段翻译、质量检查、HTML 组装、OSS 部署、飞书多维表格写回”拆成可重复执行的脚本步骤。

仓库主体是一个 Claude/Codex skill 包：`SKILL.md` 定义研究流程和质量要求，`scripts/` 提供确定性操作脚本，`references/` 提供分模块检索指南，`templates/` 提供报告 HTML 模板。

## Features

- 品牌研究项目初始化脚手架
- URL 来源清单和分模块质量门槛
- 基于 x-reader 或 Jina Reader 的来源抓取
- 原文 Markdown 保存和翻译插入脚本
- 分步骤质量检查：来源覆盖、抓取完整性、翻译质量和最终 HTML
- 单页 HTML 报告组装
- 阿里云 OSS 部署脚本
- 飞书多维表格读取待调研品牌、写回完成状态的批处理流程
- 批量调研主循环，支持锁文件、超时、dry-run 和 limit

## Tech Stack

- Python 3
- Bash
- Feishu/Lark Bitable OpenAPI
- Jina Reader fallback
- Optional `x-reader` integration
- Optional Aliyun OSS deployment through `oss2`

## Project Structure

```text
.
├── SKILL.md                    # Skill workflow and operating rules
├── research-loop.sh            # Batch research loop for pending brands
├── references/                 # Search methodology, module guides, audit prompts
├── scripts/
│   ├── init.py                 # Create a brand research artifact directory
│   ├── fetch_source.py         # Fetch one URL or a batch from source-inventory.json
│   ├── save_source.py          # Save fetched source Markdown
│   ├── add_translation.py      # Insert Chinese translation into a source file
│   ├── clean_content.py        # Remove noisy fetched content
│   ├── check_quality.py        # Validate each workflow stage
│   ├── assemble_single.py      # Build final single-page HTML
│   ├── audit_report.py         # Check generated report quality
│   ├── deploy_oss.py           # Upload final report to OSS
│   ├── bitable_read.py         # Read pending brands from Feishu Bitable
│   └── bitable_write.py        # Write completion data back to Feishu Bitable
└── templates/
    ├── single-page-template.html
    └── appendix-template.html
```

## Requirements

```bash
pip install oss2 requests
```

Optional dependency for broader source extraction:

```bash
pip install -e /path/to/x-reader
```

Feishu and OSS operations require local configuration or environment variables. Do not commit credentials.

## Getting Started

Initialize a research artifact directory:

```bash
python3 scripts/init.py "Taifun" "Taifun" \
  --company "Taifun-Tofu GmbH" \
  --country "Germany" \
  --category "Organic Tofu" \
  --output-dir outputs/taifun
```

The initializer creates the requested artifact directory:

```text
outputs/<brand>/
├── manifest.json
├── source-inventory.json
├── sources/
├── scripts/
├── templates/
└── final/
```

If `--output-dir` is omitted, `init.py` writes to the default output location relative to the installed skill directory.

## Workflow

1. Search and register source URLs in `source-inventory.json`.
2. Check source coverage:

   ```bash
   python3 scripts/check_quality.py <artifact_root> --step 1
   ```

3. Fetch sources:

   ```bash
   python3 scripts/fetch_source.py <artifact_root> --batch
   ```

4. Add translations with `scripts/add_translation.py`.
5. Validate translation quality:

   ```bash
   python3 scripts/check_quality.py <artifact_root> --step 3
   ```

6. Clean, assemble, and deploy:

   ```bash
   python3 scripts/clean_content.py <artifact_root>
   python3 scripts/assemble_single.py <artifact_root>
   python3 scripts/deploy_oss.py <artifact_root>
   ```

7. Audit the generated report:

   ```bash
   python3 scripts/audit_report.py <artifact_root>
   ```

## Batch Research Loop

`research-loop.sh` reads the next pending brand from Feishu Bitable, starts an isolated Claude session, validates artifacts, and writes completion data back.

```bash
bash research-loop.sh
bash research-loop.sh --limit 10
bash research-loop.sh --dry-run --limit 3
```

The loop includes local lock files, timeout handling, completion validation, idempotent write-back, and per-brand logs.

## Configuration

Common configuration sources:

- `bitable-config.json` for Feishu app/table settings
- `FEISHU_APP_ID` and `FEISHU_APP_SECRET` when no config file is present
- `oss-config.json` inside an artifact directory for OSS deployment

Keep all credentials outside version control.

## Notes

- The scripts intentionally separate search, fetching, translation, assembly, and audit stages.
- `scripts/` and `templates/` are copied into each artifact directory so a research run can keep a locked tool/template version.
- The generated report is expected at `<artifact_root>/final/report.html`.
