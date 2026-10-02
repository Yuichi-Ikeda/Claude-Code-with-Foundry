# Claude Code から Microsoft Foundry の Claude モデルを利用する

## はじめに

開発者個人が Claude Code から Microsoft Foundry の Claude モデルを直接利用する場合は、[Microsoft の公式ドキュメント](https://learn.microsoft.com/azure/foundry/foundry-models/how-to/configure-claude-code) を参照してください。

本資料は、**IT 基盤部門による組織への導入**を想定しています。[Azure API Management](https://azure.microsoft.com/ja-jp/products/api-management)（以下、APIM）を AI Gateway として利用し、ユーザー認証と利用権限の制御を行う構成、および利用状況を監査する際の注意点を紹介します。

## 認証パターン

APIM で誰にアクセスを許可するかによって、次の 3 パターンを用意しています。いずれの構成でも、APIM は **検証に成功した要求だけ** を **APIM 自身のマネージド ID** で Foundry に転送します。利用者の資格情報が Foundry に渡ることはなく、利用者個人に Foundry リソースの Azure RBAC 権限を付与する必要もありません。

| # | 認証パターン | 識別できる単位 | 利用者の操作 |
| --- | --- | --- | --- |
| 1 | [APIM キー認証](APIM_KEY.md) | キーの所有者（個人は識別しない） | キーを設定するだけ |
| 2 | [Entra ID 認証](APIM_Entra_ID.md) | Entra ID のユーザー個人 | `az login` でサインイン |
| 3 | [Entra ID 認証 + APIM キー認証](APIM_Entra_ID_KEY.md) | ユーザー個人とキーの所有者の両方 | `az login` でサインイン + キーを設定 |

### 1. APIM サブスクリプション キー認証

Claude Code が各要求に付けるサブスクリプション キー（`ANTHROPIC_API_KEY`）を APIM が検証します。Entra ID の設定が不要で、最も少ない手順で始められます。一方で、識別できるのは **キーの所有者** までであり、条件付きアクセスや多要素認証、ユーザー・グループ単位の利用権限は適用されません。利用状況を集計できる単位も APIM のサブスクリプションまでです。検証環境や、チーム単位でキーを払い出して利用を分離する用途に向いています。

### 2. Entra ID 認証

利用者が `az login` でサインインし、Claude Code が Azure CLI 経由で取得したアクセス トークンを APIM が検証します。アプリ ロール（`Claude.User`）による利用権限の制御、条件付きアクセスや多要素認証の適用、ユーザー単位の監査ができます。組織への標準的な展開に向いています。

### 3. Entra ID 認証 + APIM サブスクリプション キー認証

アクセス トークンとサブスクリプション キーの両方を APIM が検証し、**どちらか一方だけでは利用できません**。ユーザー単位の制御に加えて、サブスクリプション単位のレート制限・クォータや、キーごとの Foundry バックエンドの振り分けを組み合わせたい場合に向いています。

> [!TIP]
> 組織への導入では、ユーザー単位の識別と権限制御ができるパターン 2 を基本に検討してください。チームや用途ごとに上限や接続先を分ける必要がある場合はパターン 3、ユーザー単位の識別が不要な検証用途であればパターン 1 を選択します。

## 利用状況の監視

[カスタムメトリックによる LLM トークン使用量の監視](LLM_Logging.md) では、APIM の [llm-emit-token-metric ポリシー](https://learn.microsoft.com/azure/api-management/llm-emit-token-metric-policy) を使って、Claude モデルのトークン使用量を Application Insights にカスタムメトリックとして記録します。記録したデータは、Entra ID のユーザー（UPN）とモデルごとに KQL で集計できます。ユーザーの識別には APIM で検証したアクセス トークンを使うため、認証パターン 2 または 3 の構成が前提です。