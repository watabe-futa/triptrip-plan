# TripTrip Business & IT Strategic Planning Repository

## Overview

This repository contains comprehensive business and IT strategic planning documents for the TripTrip service - an innovative mobile travel application platform. The project aims to create world-class strategic documentation totaling over 200,000 lines (100,000+ lines for business strategy, 100,000+ lines for IT strategy).

## 毎日の作業開始方法（スラッシュコマンド版）

### 利用可能なスラッシュコマンド

| コマンド | 説明 | 使用タイミング |
|---------|------|--------------|
| `/coordinator` | コーディネーターを起動し、本日の2-4個のプロンプトを生成 | 毎朝の作業開始時 |
| `/progress` | 現在の進捗状況を確認（完了文書数、行数、進捗率） | 進捗確認時 |
| `/update-index` | 新規作成文書を検出してインデックスを自動更新 | 文書作成後 |

### 作業フロー

1. **朝の作業開始**
   - Claudeで `/coordinator` と入力
   - 生成された2-4個のプロンプトを確認

2. **文書作成**
   - 各プロンプトを新しいClaudeウィンドウにコピペ
   - 並列で実行（各エージェントが独立して1,200-1,500行の文書を作成）

3. **進捗更新**
   - 文書作成後、`/update-index` で自動更新
   - `/progress` で現在の進捗を確認

### 従来の方法（スラッシュコマンドが使えない場合）

<details>
<summary>クリックして展開</summary>

新しいClaudeウィンドウで以下のプロンプトを入力：

```
あなたはTripTrip戦略計画のコーディネーターエージェントです。
agent-prompts/coordinator-agent.mdの指示に従い、
progress-tracking/document-index.mdの進捗を確認して、
本日実行すべき2-4個の具体的なプロンプトを生成してください。
各プロンプトはコピペして別のClaudeウィンドウで実行できる形式で出力してください。
```

</details>

## Project Structure

- **business-strategy/** - Comprehensive business planning documents
- **it-strategy/** - Detailed IT architecture and implementation plans
- **agent-prompts/** - AI agent coordination prompts
- **progress-tracking/** - Project progress monitoring

## Methodology

This project employs parallel AI agent processing with 3-5 specialized agents working simultaneously:
- Strategy Agent - Market analysis and competitive strategy
- Business Model Agent - Revenue models and value propositions
- Technical Architecture Agent - System design and infrastructure
- Implementation Agent - Development roadmaps and resource planning
- Finance & Risk Agent - Financial modeling and risk assessment

## Documentation Standards

- Each document: 1,200-1,500 lines for optimal AI readability
- **Language: 日本語（全文書）**
- McKinsey/BCG-level strategic depth
- Accenture/IBM-level technical precision
- Global scalability focus

## Status

**Active Development** - Targeting completion of 200,000+ lines of strategic documentation

## Related Repositories

- **triptrip** - Frontend mobile application implementation

---

*Leveraging world-class consulting methodologies and AI-powered document generation*