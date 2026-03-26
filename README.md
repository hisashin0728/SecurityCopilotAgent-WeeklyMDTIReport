# Weekly Threat Intelligence Report

> Microsoft Threat Intelligence と Microsoft Defender を組み合わせて、過去1週間に公開・更新された脅威情報を業界別に整理し、CVE を伴う攻撃が自社環境に与える影響を評価して、CISO 向け HTML レポートをメール送信する Security Copilot Agent ソリューション。

## 概要

`WeeklyThreatIntelReport` は、Microsoft Security Copilot の Standard Agent として動作し、以下を自動化します。

- Microsoft Threat Intelligence プラグインを使った過去1週間の脅威情報収集
- 業種・業界に基づく脅威情報のフィルタ
- IOC、脅威アクター、脆弱性悪用、攻撃キャンペーンの把握
- CVE を伴う脅威について Defender から影響資産を確認
- CISO 向けの要点整理と推奨対策を含む HTML メール配信

このソリューションは、SOC の詳細な運用レポートではなく、経営層やセキュリティ責任者が「今週、何を優先的に把握・判断すべきか」を短時間で理解できることを目的としています。

## アーキテクチャ

```
┌──────────────────────────────┐
│ Microsoft Security Copilot   │
│ Standard Agent               │
│ WeeklyThreatIntelReport      │
└──────────────┬───────────────┘
               │
               ├──────────────▶ ThreatIntelligence.DTI
               │                - 業界関連の脅威記事
               │                - CVE 情報
               │
               ├──────────────▶ Defender KQL
               │                - CVE の影響資産
               │                - デバイス / ソフトウェア / 露出レベル
               │
               └──────────────▶ Azure Logic App
                                - HTML メール送信
```

## 含まれるファイル

| ファイル | 説明 |
|---------|------|
| `WeeklyThreatIntelReport.yaml` | Security Copilot Agent マニフェスト |
| `WeeklyThreatIntelReport_card.html` | プラグインカード（ビジュアル概要） |
| `WeeklyThreatIntelReport_LogicApp_ARM.json` | HTML メール送信用 Logic App ARM テンプレート |
| `WeeklyThreatIntelReport_README.md` | この README |

## 前提条件

- **Microsoft Security Copilot** を利用可能であること
- **Microsoft Defender XDR / Defender for Endpoint** の Advanced Hunting が利用可能であること
- **ThreatIntelligence.DTI** スキルセットが利用可能であること
- **Azure サブスクリプション** があり、Logic App をデプロイできること
- **Office 365 Outlook コネクタ** を Logic App で利用できること

## セットアップ手順

### 1. Logic App のデプロイ

まずメール送信に使用する Logic App をデプロイします。

```powershell
az deployment group create `
  --resource-group <リソースグループ名> `
  --template-file WeeklyThreatIntelReport_LogicApp_ARM.json `
  --parameters emailAddress="ciso@example.com"
```

デプロイ後に Azure Portal で `office365` 接続を開き、必要に応じて認証を完了してください。

### 2. Security Copilot Agent のアップロード

1. Security Copilot ポータルを開く
2. `Settings` → `Custom plugins` → `Add plugin`
3. `WeeklyThreatIntelReport.yaml` をアップロード
4. プラグインを有効化

### 3. Agent セットアップ時の入力値

セットアップ時に次の値を入力します。

| パラメータ | 説明 | 例 |
|-----------|------|----|
| `LogicAppSubscriptionId` | Logic App を配置した Azure サブスクリプション ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `LogicAppResourceGroup` | Logic App を配置したリソースグループ名 | `rg-securitycopilot` |
| `LogicAppWorkflowName` | Logic App のワークフロー名 | `PluginLogicApp_WeeklyThreatIntelReport` |
| `Industry` | 脅威情報を絞り込む対象業界 | `Financial` |

## Agent の構成

### スキル一覧

| スキル名 | 形式 | 説明 |
|---------|------|------|
| `GenerateThreatIntelReport` | Agent | レポート生成とメール送信を統括するメインスキル |
| `FindThreatIntelligence` | ThreatIntelligence.DTI | 過去1週間の脅威記事や攻撃情報を取得 |
| `GetCvesByIdsDti` | ThreatIntelligence.DTI | CVE の概要と影響技術を取得 |
| `GetAssetsImpactedByCve` | KQL (Defender) | 自社環境で CVE の影響を受けるデバイスとソフトウェアを取得 |
| `SendThreatReportEmail` | LogicApp | 生成した HTML レポートをメール送信 |

### ワークフロー

```
1. 過去1週間の MDTI 脅威情報を取得
2. 業界に関連する脅威を抽出
3. CVE を含む脅威を特定
4. Defender で自社影響資産を確認
5. CISO 向けに重要脅威と推奨対策を要約
6. HTML レポートを生成
7. Logic App でメール送信
```

## レポート内容

レポートは CISO 向けに、次の観点で整理されます。

- **エグゼクティブサマリー**
  - 今週の主要脅威件数
  - 重大な脅威件数
  - 自社に影響する CVE 件数
  - 経営層向けの要点

- **最新脅威の一覧**
  - 脅威名 / アクター / キャンペーン
  - 関連 CVE
  - 自社影響資産
  - 脅威の概要

- **重大な脅威の強調表示**
  - 業界関連性
  - 実悪用状況
  - 自社影響の有無
  - 推奨対策

- **監視すべき脅威**
  - 現時点では影響が限定的だが継続監視が必要なもの

- **CISO 向け推奨事項**
  - 直近の優先対応事項
  - 経営層に共有すべきリスク
  - IT / SOC / 脆弱性管理チームへのアクション

## Logic App の入力スキーマ

`SendThreatReportEmail` から Logic App に渡される JSON は以下です。

```json
{
  "ReportHtml": "<html>...生成されたHTMLレポート全文...</html>"
}
```

## Defender KQL の役割

`GetAssetsImpactedByCve` は、指定された CVE について以下を取得します。

- 影響を受けるデバイス名
- 影響ソフトウェア名 / バージョン
- 推奨更新プログラム
- CVSS スコア
- エクスプロイト可用性
- 露出レベル
- インターネット公開状態
- センサー健全性

これにより、Threat Intelligence で把握した攻撃情報が「自社にとって本当に重要か」を具体的に判断できます。

## 想定ユースケース

- 金融業界向けの週次脅威ブリーフィング
- 経営会議向けのサイバーリスク報告
- CISO / セキュリティ責任者向けの優先順位付け支援
- 脅威インテリジェンスと脆弱性管理の橋渡し

## トラブルシューティング

| 症状 | 原因候補 | 対処 |
|------|----------|------|
| メールが届かない | Logic App の Office 365 接続未認証 | Azure Portal で API 接続を承認 |
| Logic App が起動しない | SubscriptionId / ResourceGroup / WorkflowName の設定誤り | Agent セットアップ値を再確認 |
| CVE の影響資産が 0 件 | 自社環境に該当ソフトウェアが存在しない | 正常動作の可能性あり。レポートには影響なしとして記載 |
| 脅威情報が想定より少ない | 業界フィルタが狭い / 過去1週間に該当情報が少ない | Industry を広めに指定して比較 |
| 英語の説明が混ざる | DTI の返却データが英語 | Agent Instructions で日本語要約する構成になっているか確認 |

## カスタマイズ例

- 集計期間を 1 週間から 30 日へ拡張
- 業界パラメータを複数指定対応へ変更
- CISO 向けに加えて SOC 向け詳細レポートを追加
- Teams 通知や SharePoint 配布に変更

## ライセンス

MIT
