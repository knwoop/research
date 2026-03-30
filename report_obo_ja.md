# OBO（On-Behalf-Of）: AIエージェント権限委譲における技術標準

## エグゼクティブサマリー

On-Behalf-Of（OBO）は、AIエージェント認可の基盤パターンとして急速に台頭している。もともとOAuth 2.0 Token Exchange（RFC 8693）でサービス間委譲のために定義されたOBOは、現在IETF Internet-Draft、プラットフォーム固有の実装（Microsoft Entra Agent ID）、および補完プロトコル（Google A2A、MCP、GNAP）を通じて、AIエージェント向けに積極的に拡張されている。本レポートでは、OBOメカニズムがAIエージェントの権限委譲特有の課題 — マルチホップ委譲チェーン、スコープ減衰、Confused Deputy問題、監査可能性 — にどのように適応されているかを技術的に深く分析し、2026年初頭時点の新興標準の全体像を調査する。

## 目次

1. [基礎: RFC 8693と従来のOBO](#1-基礎-rfc-8693と従来のobo)
2. [IETF Draft: AIエージェント向けOAuth 2.0 OBO拡張](#2-ietf-draft-aiエージェント向けoauth-20-obo拡張)
3. [Microsoft Entra Agent ID: 実践におけるOBO](#3-microsoft-entra-agent-id-実践におけるobo)
4. [委譲トークンの仕組み: クレーム、チェーン、コンポジット](#4-委譲トークンの仕組み-クレームチェーンコンポジット)
5. [マルチエージェント委譲パターン](#5-マルチエージェント委譲パターン)
6. [補完プロトコルと標準規格](#6-補完プロトコルと標準規格)
7. [代替トークン形式: Macaroons、Biscuits、オフライン減衰](#7-代替トークン形式-macaroonsbiscuitsオフライン減衰)
8. [セキュリティ上の考慮事項と未解決の問題](#8-セキュリティ上の考慮事項と未解決の問題)
9. [結論](#9-結論)
10. [出典](#10-出典)

---

## 1. 基礎: RFC 8693と従来のOBO

### 1.1 OAuth 2.0 Token Exchange

<a href="https://datatracker.ietf.org/doc/html/rfc8693" target="_blank">RFC 8693</a>は、OAuth 2.0認可サーバーからセキュリティトークンを要求・取得するためのプロトコルを定義し、OBO委譲の基礎的なセマンティクスを確立した。この仕様はSecurity Token Service（STS）モデルを導入し、グラントタイプ`urn:ietf:params:oauth:grant-type:token-exchange`を使用する。

### 1.2 委譲（Delegation）vs. 偽装（Impersonation）

RFC 8693は2つの認可セマンティクスの重要な区別を定義している：

**偽装（Impersonation）**: プリンシパルAがプリンシパルBの全権限を受け取り、Bと区別がつかなくなる。リソースサーバーはBのIDのみを認識する。APIゲートウェイでのレガシーシステム向けトークン交換やサポートスタッフによるユーザー問題の再現などが使用例となる。

**委譲（Delegation）**: プリンシパルAが自身のIDを保持しつつ、明示的にBの代理として行動する。両方のIDが「コンポジットトークン」に保持される。RFCが述べるように：「プリンシパルAはBとは別の独自のIDを持ち続け、BがAにいくつかの権利を委譲した可能性があるものの、すべての行動はAがBを代表して行っていることが明示的に理解される。」

この区別はAIエージェントにとって根本的に重要である：リソースサーバーと監査システムは、誰がアクションを*認可*したか（ユーザー）と誰がそれを*実行*したか（エージェント）の*両方*を知る必要があるため、委譲（偽装ではない）が正しいセマンティクスとなる。

### 1.3 リクエストパラメータ

トークン交換リクエストは以下の主要パラメータを使用する：

| パラメータ | 必須 | 説明 |
|-----------|------|------|
| `subject_token` | はい | 要求が代理で行われる当事者を表す |
| `subject_token_type` | はい | サブジェクトトークンの形式を識別（JWT、SAMLなど） |
| `actor_token` | いいえ | 発行されたトークンの使用を認可された行為者を表す |
| `actor_token_type` | 条件付き | `actor_token`が存在する場合に必須 |
| `scope` | いいえ | 新しいトークンに要求するスコープ |
| `audience` | いいえ | 新しいトークンの対象サービス |

### 1.4 `act`クレームと`may_act`クレーム

RFC 8693は委譲専用の2つのJWTクレームを導入している：

- **`act`（actor）**: 委譲シナリオにおける行為者を識別する。「JWT内で委譲が発生したことを表現し、権限が委譲された行為者を識別する手段を提供する。」

- **`may_act`（authorized actor）**: サブジェクトトークンに埋め込まれた認可アサーションで、特定の当事者がサブジェクトの代理として行動する事前認可を持つことを宣言する。これにより、認可サーバーは追加の帯域外チェックなしに委譲リクエストを検証できる。

---

## 2. IETF Draft: AIエージェント向けOAuth 2.0 OBO拡張

### 2.1 概要

AIエージェントのOBOを対象とする最も重要な標準化活動は、<a href="https://datatracker.ietf.org/doc/draft-oauth-ai-agents-on-behalf-of-user/" target="_blank">draft-oauth-ai-agents-on-behalf-of-user</a>（現在バージョン02）であり、WSO2のT. S. SenarathとA. Dissanayakaによって執筆されている。このInternet-Draftは、OAuth 2.0 Authorization Code Grantフローを拡張し、AIエージェント専用のユーザー委任認可を実現する。

### 2.2 なぜ新しい拡張が必要か？

標準のOAuth 2.0フローはエージェント委譲を十分にカバーしていない：

- **Authorization Code Grant**: クライアントアプリケーションは識別するが、その中で動作する特定のエージェントは識別しない。ユーザーは*アプリケーション*に同意するが、*エージェント*には同意しない。
- **Client Credentials Grant**: ユーザー同意メカニズムを提供しない — エージェントはアプリケーションレベルの権限で動作する。
- **RFC 8693 Token Exchange**: フロントチャネルを通じてエージェントに対する明示的なユーザー同意を取得するためのネイティブサポートがない。ユーザーインタラクティブフローを通じて初期サブジェクトトークンを取得する方法を規定していない。

### 2.3 新しいパラメータ

ドラフトは2つのパラメータを導入する：

**`requested_actor`**（認可エンドポイント）: クライアントが委譲アクセスを要求するAIエージェントの一意識別子。認可リクエストでREQUIREDであり、認可サーバーがエージェント固有の同意画面を表示できるようにする。

**`actor_token`**（トークンエンドポイント）: エージェントに発行された有効なトークンで、認可コード交換時にエージェントを認証する。エージェントを識別する`sub`クレームを含む必要がある（MUST）。

### 2.4 プロトコルフロー

完全な10ステップのフロー：

```
┌──────┐  ┌──────────┐  ┌────────────┐  ┌─────────────┐  ┌──────────────┐
│ ユーザー│  │ AIエージェント│  │  クライアント │  │  認可サーバー  │  │ リソースサーバー │
└──┬───┘  └────┬─────┘  └─────┬──────┘  └──────┬──────┘  └──────┬───────┘
   │           │              │                 │                │
   │           │  1. エージェントが行動の必要性を通知  │                │
   │           │──────────────>                 │                │
   │           │              │  2. リソースアクセス              │
   │           │              │─────────────────────────────────>│
   │           │              │  3. 401/403 + WWW-Authenticate   │
   │           │              │<─────────────────────────────────│
   │  4. requested_actor + PKCEでリダイレクト    │                │
   │<─────────────────────────│                 │                │
   │  5. 認証 + エージェント+スコープへの同意     │                │
   │─────────────────────────────────────────── >                │
   │           │              │  6. 認可コード                    │
   │           │              │<────────────────│                │
   │           │              │  7. コード + PKCEベリファイア +    │
   │           │              │     actor_token                  │
   │           │              │────────────────>│                │
   │           │              │  8. コード、PKCE、               │
   │           │              │     actor_tokenの検証             │
   │           │              │  9. OBOアクセストークン発行        │
   │           │              │<────────────────│                │
   │           │              │  10. OBOトークンでリソースアクセス  │
   │           │              │─────────────────────────────────>│
```

主要な革新点：
- フローは**リソースサーバーのチャレンジ**（HTTP 401/403）によってトリガーでき、動的な同意取得が可能。
- 認可コードは（ユーザー、クライアント、アクター）の三つ組みにバインドされ、（ユーザー、クライアント）だけではない。
- PKCEはすべてのフローでREQUIRED。

### 2.5 アクセストークンのクレーム

発行されるJWTには以下が含まれる：

```json
{
  "iss": "https://authorization-server.com/oauth2/token",
  "sub": "user-456",
  "azp": "s6BhdRkqt3",
  "act": { "sub": "actor-finance-v1" },
  "scope": "read:email write:calendar",
  "exp": 1746009896,
  "iat": 1745923496,
  "jti": "unique-token-id"
}
```

`sub`クレームはユーザーを識別。`act.sub`クレームはエージェントを識別。`azp`クレームはクライアントアプリケーションを識別。これにより**明確で監査可能な委譲チェーン**が提供される：ユーザー → クライアント → エージェント → リソース。

### 2.6 関連するIETF Draft

AIエージェント認可の隣接する側面に取り組む他のドラフト：

- **<a href="https://datatracker.ietf.org/doc/draft-klrc-aiagent-auth/" target="_blank">draft-klrc-aiagent-auth-00</a>**: SPIFFE/WIMSE識別子、mTLS、OAuth 2.0を組み合わせたエージェント認証・認可の包括的フレームワーク。レイヤードアテステーションによるAgent Identity Management System（AIMS）を定義。
- **<a href="https://www.ietf.org/archive/id/draft-rosenberg-oauth-aauth-00.html" target="_blank">draft-rosenberg-oauth-aauth-00（AAuth）</a>**: 音声/テキストチャネルを通じてユーザーからPIIを収集するAIエージェント向けOAuth 2.1拡張。リダイレクトフローではなく、ユーザーID検証を通じてトークンを取得。
- **<a href="https://datatracker.ietf.org/doc/html/draft-liu-oauth-a2a-profile-00" target="_blank">draft-liu-oauth-a2a-profile-00</a>**: エージェント間コールチェーンコンテキストを短命のトランザクショントークンに埋め込むA2Aプロファイル。
- **<a href="https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-chaining/" target="_blank">draft-ietf-oauth-identity-chaining-08</a>**: 信頼ドメイン境界を越えてIDと認可を保持するメカニズムを定義 — マルチクラウドエージェントデプロイメントに不可欠。

---

## 3. Microsoft Entra Agent ID: 実践におけるOBO

### 3.1 エージェントIDアーキテクチャ

<a href="https://learn.microsoft.com/en-us/entra/agent-id/identity-platform/agent-on-behalf-of-oauth-flow" target="_blank">Microsoft Entra Agent ID</a>は、AIエージェント向けOBOの最初の主要プラットフォーム実装であり、2つの重要な概念を導入している：

- **Agent Identity Blueprint（エージェントIDブループリント）**: エージェントの機能と委譲された権限を定義する親アプリケーション登録。OAuthリソース（API）アプリケーションとして機能する。
- **Agent Identity（エージェントID）**: ブループリントの子インスタンスで、特定の実行中のエージェントを表す。`InheritDelegatedPermissions`プロパティを通じて親から委譲された権限を継承する。

### 3.2 プロトコルステップ

フローは標準のOAuth 2.0 OBOをマネージドIDクレデンシャルレイヤーで拡張する：

1. **ユーザーが認証**し、クライアントでアクセストークン（Tc）を取得。
2. **クライアントがTcを送信**し、Agent Identity Blueprintへ。
3. **Agent Identity Blueprint**がマネージドIDクレデンシャル（TUAMI）を提示して交換トークンを要求：
   ```
   POST /oauth2/v2.0/token
   client_id=AgentBlueprint
   &scope=api://AzureADTokenExchange/.default
   &fmi_path=AgentIdentity
   &client_assertion=TUAMI
   &grant_type=client_credentials
   ```
   これによりT1（エージェントを識別するトークン）が返される。
4. **Agent Identity**がT1とTcの両方を使用してOBO交換を実行：
   ```
   POST /oauth2/v2.0/token
   client_id=AgentIdentity
   &scope=https://resource.example.com/scope1
   &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
   &client_assertion={T1}
   &grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
   &assertion={Tc}
   &requested_token_use=on_behalf_of
   ```
5. **Entra IDが両トークンを検証**し（T1のaudienceはAgent Identity Blueprint client IDと一致する必要がある）、リソーストークンを発行。

### 3.3 主要な設計判断

- **マネージドID**が推奨されるクレデンシャルタイプであり、自動ローテーションと安全なストレージを提供。
- **クライアントシークレットは本番環境では明示的に非推奨**。
- **権限継承**はブループリントからIDへの境界を越えて動作し、マルチインスタンスデプロイメントの同意の複雑さを軽減するが、テナント境界内に制限される。
- **リフレッシュトークンのサポート**により、非同期およびバックグラウンド処理シナリオが可能。

---

## 4. 委譲トークンの仕組み: クレーム、チェーン、コンポジット

### 4.1 コンポジットトークン構造

委譲トークン（「コンポジットトークン」）は、サブジェクト（ユーザー）とアクター（エージェント）の両方をエンコードする。基本構造：

```json
{
  "sub": "user@example.com",
  "act": {
    "sub": "agent:infrabot"
  },
  "aud": "api.github.com",
  "scope": "read:repos",
  "exp": 1721890041
}
```

`sub`クレームは「誰の権限で？」に回答。`act.sub`クレームは「誰が行動しているか？」に回答。`aud`クレームはトークンの有効範囲を制限。`scope`は許可されるアクションを制限。

### 4.2 マルチホップチェーンのためのネストされた`act`クレーム

エージェントが他のエージェントに委譲する場合、`act`クレームは再帰的にネストされる：

```json
{
  "sub": "user:sam",
  "act": {
    "sub": "agent:market-analysis",
    "act": {
      "sub": "agent:supply-chain",
      "act": {
        "sub": "agent:data-retriever"
      }
    }
  }
}
```

RFC 8693によると：「最も外側の`act`クレームが現在のアクターを表し、ネストされた`act`クレームは以前のアクターを表す。」重要なのは、**アクセス制御の判断には、トップレベルのクレームと現在の（最も外側の）アクターのみを考慮すべき**であるということだ。ネストされたアクターは認可入力ではなく、監査証跡として機能する。

### 4.3 SPIFFEベースのアクター識別

クラウドネイティブデプロイメントでは、アクター識別子に<a href="https://spiffe.io/docs/latest/spire-about/spire-concepts/" target="_blank">SPIFFE</a> URIを使用でき、アテステーションにバインドされたIDを提供する：

```json
{
  "act": {
    "sub": "spiffe://cluster.local/ns/default/sa/market-analysis-agent",
    "act": {
      "sub": "spiffe://cluster.local/ns/default/sa/supply-chain-agent"
    }
  },
  "act_depth": 2,
  "sub": "user-uuid-here",
  "typ": "OBO"
}
```

SPIFFEはワークロードアテステーション（単なる登録ではなく実行環境に基づくID導出）、短命のクレデンシャル（SVID）、信頼ドメイン間のフェデレーションを提供する — すべてスケールでのエージェントIDに不可欠なプロパティである。

---

## 5. マルチエージェント委譲パターン

### 5.1 スコープ減衰（Scope Attenuation）

OBO委譲の基本原則は、**各ホップで権限を縮小し、決して拡大しない**ことである。結果として得られるトークンは以下の3つの交差点を表す：

1. ユーザーの権限
2. エージェント/サービスに設定された権限
3. ターゲットリソースの受け入れポリシー

<a href="https://blog.christianposta.com/explaining-on-behalf-of-for-ai-agents/" target="_blank">Christian Posta</a>が説明するように、トークン交換は「権限の増幅ではなく、削減」を保証する — 各ホップは特定のaudienceと縮小されたスコープを持つ、より制約されたトークンを生成する。

減衰パターンの例：
```
ユーザートークン:      {identity, expiry, buy_stock, market_data, email}
  ↓ エージェントA OBO: {identity, expiry, buy_stock, market_data}  [email削除]
    ↓ エージェントB OBO: {identity, expiry, market_data}           [buy_stock削除]
```

### 5.2 トークンブローカーアーキテクチャ

<a href="https://www.garvik.dev/ai/on-behalf-of-token-complex" target="_blank">エンタープライズ実装</a>では通常、トークンブローカー仲介者を導入する：

1. ユーザーが認証し、標準アクセストークンを受け取る。
2. アプリケーションがこのトークンをトークンブローカーと交換し、スコープされたOBOトークンを取得。
3. AIエージェントはより限定された短命のOBOトークンを受け取る — 元のユーザートークンは決して受け取らない。
4. MCPサーバーまたはリソースサーバーがアクション実行前にOBOトークンを検証。

これにより、エージェントがサービス全体でユーザーの全権限を継承する「ゴッドモード」シナリオを防止する。

### 5.3 コールチェーン向けトランザクショントークン

<a href="https://datatracker.ietf.org/doc/html/draft-liu-oauth-a2a-profile-00" target="_blank">OAuth Transaction TokensのA2Aプロファイル</a>は、コールチェーンコンテキストを短命トークンに埋め込むことを提案する：

- **`tctx`（トランザクションコンテキスト）**: 元のユーザー入力からの不変値で、チェーン全体を通じて変更されずに保持される。
- **`rctx`（リクエストコンテキスト）**: 各ホップでエージェントが追加する可変コンテキスト（観察、決定、メタデータ）。
- **`purp`（目的）**: 委譲チェーン内のすべてのトークンをリンクするタスク識別子。

これらのトランザクショントークンは、分以下のライフタイムとスコープ拡張を防ぐ不変のコールコンテキストを通じて、「最小権限の保証と監査可能性」を提供する。

---

## 6. 補完プロトコルと標準規格

### 6.1 Google Agent2Agent（A2A）プロトコル

<a href="https://a2a-protocol.org/latest/specification/" target="_blank">A2Aプロトコル</a>は相互運用可能なエージェント通信を実現するが、OBOとは異なるアプローチで委譲に取り組む：

- **Agent Cards**はサポートする認証スキーム（API Key、OAuth 2.0、OIDC、mTLS）を宣言。
- **`AUTH_REQUIRED`タスク状態**により、追加の権限が必要な場合にエージェントが認可責任をクライアントに委譲可能。
- A2Aはエージェント間委譲のための直接的なOBOパターンを**実装していない**。認証はクライアント-サーバー間のまま。
- <a href="https://github.com/a2aproject/A2A/issues/19" target="_blank">Issue #19</a>が「Agent2Agentサーバーの委譲ユーザー認可」をオープンな設計課題として明示的に追跡。

### 6.2 Model Context Protocol（MCP）

<a href="https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization" target="_blank">MCPの認可モデル</a>はリモートサーバーにOAuth 2.1 + OIDCを必須としている：

- トークン検証は**すべてのツール呼び出し時**に行われ、セッション確立時だけではない。
- MCPサーバーはサードパーティ認可サーバーに対してOAuthクライアントとして機能しつつ、MCPクライアントに対してはOAuth認可サーバーとして機能する — デュアルロール委譲パターン。
- マルチホップチェーン（ユーザー → AIホスト → MCPクライアント → MCPサーバー → バックエンドAPI）は各境界で認可コンテキストを保持する必要がある。
- <a href="https://www.flowhunt.io/blog/mcp-authentication-authorization-oauth-confused-deputy/" target="_blank">MCPは明示的にOBOを使用</a>してConfused Deputy問題を防止する：ユーザートークンを転送するのではなく、特定の操作にスコープされたサーバー発行のOBOトークンと交換する。

### 6.3 GNAP（RFC 9635）

<a href="https://www.rfc-editor.org/rfc/rfc9635.html" target="_blank">GNAP（Grant Negotiation and Authorization Protocol）</a>は2024年10月にRFC 9635として正式化され、エージェント委譲においてOAuth 2.0より優れたアーキテクチャ上の利点を提供する：

- **ロール分離**: GNAPはリソースオーナーとエンドユーザーを分離し、「エージェントがユーザーが所有するアクセスを要求する」ことをより明確にモデリング。
- **事前登録不要**: エージェントはOAuthクライアントとして事前登録する必要がなく、動的にグラントを交渉。
- **グラントあたり複数のアクセストークン**: 単一のグラントリクエストで複数のリソース向けトークンを取得可能。
- **継続グラント**: 長期の認可関係を再同意なしに時間をかけて再交渉可能。
- **JSONネイティブ**: 構造化されたリクエスト/レスポンスボディ（OAuthのフォームエンコードパラメータと比較）がマシン間通信に適している。

ただし、GNAPの採用はまだ初期段階 — 2026年初頭時点で<a href="https://ciamweekly.substack.com/p/gnap-for-mcp-authorization" target="_blank">知られている実装は約10件</a>にとどまる。

### 6.4 WIMSE（Workload Identity in Multi-System Environments）

<a href="https://datatracker.ietf.org/wg/wimse/about/" target="_blank">IETF WIMSEワーキンググループ</a>はクラウドとプラットフォーム間のワークロードIDに取り組んでいる。最近のドラフトでは明示的にAIエージェントのユースケースが追加されている：

- <a href="https://datatracker.ietf.org/doc/draft-ietf-wimse-arch/" target="_blank">draft-ietf-wimse-arch-07</a>にAI固有のシナリオ、SPIFFEベースのエージェント識別子、OAuth Identity Chainingとの整合が含まれるようになった。
- WIMSEワークロードクレデンシャルは、OBOトークンが`act`クレームで参照する暗号化IDレイヤーを提供する。

### 6.5 AuthZEN（OpenID Authorization API 1.0）

<a href="https://openid.net/wg/authzen/" target="_blank">AuthZEN</a>は2026年1月にFinal Specificationとして承認され、認可評価APIを標準化している。AIエージェントでは、すべてのツール呼び出しがAuthZEN評価にマッピングされる：

- **Subject**: エージェントのID（例：SPIFFE ID）
- **Action**: 操作（例：「send_email」）
- **Resource**: ターゲット（例：「contact_list」）
- **Context**: 委譲ユーザー、影響範囲の分類、可逆性フラグ、行動異常スコア

AuthZENは<a href="https://www.rockcybermusings.com/p/i-agent-authentication-authorization-gap" target="_blank">認証標準が残すギャップ</a>を埋める：発行時のスコープ適用ではなく、アクションごとの認可評価。

---

## 7. 代替トークン形式: Macaroons、Biscuits、オフライン減衰

### 7.1 JWTベースOBOの限界

標準のJWTトークンは、認可サーバーに連絡せずに発行後に制限することができない。分散型マルチエージェントシステムでは、これがスケールでのレイテンシと単一障害点のリスクを生む。

### 7.2 Macaroons

<a href="https://research.google/pubs/pub41892/" target="_blank">Macaroons</a>（Google Research、2014年）は、サーバー検証なしに権限を減衰させる追記専用の「caveat（条件）」を持つ認可クレデンシャルである：

- トークン保持者はローカルで署名付き制約を追加でき、権限を拡大することは決してない。
- Caveatは元の発行者まで遡って検証可能なチェーンを作成する。
- **Macaroonsは、保持者が自身の権限を削減して下流に渡すことができる唯一のクレデンシャルプリミティブ** — まさにエージェント委譲に必要なもの。

### 7.3 Biscuits

<a href="https://www.clever.cloud/blog/engineering/2021/04/12/introduction-to-biscuit/" target="_blank">Biscuits</a>は公開鍵暗号（MACベースの共有秘密と比較）を使用してMacaroonsを改善し、信頼ドメイン間で共有秘密なしに検証を可能にする。

### 7.4 Delegation Capability Tokens（DCT）

<a href="https://www.marktechpost.com/2026/02/15/google-deepmind-proposes-new-framework-for-intelligent-ai-delegation-to-secure-the-emerging-agentic-web-for-future-economies/" target="_blank">Google DeepMindは</a>Macaroon/Biscuit技術に基づくDelegation Capability Tokens（DCT）を提案し、暗号化caveatを使用してエージェント委譲チェーンで最小権限を強制する。<a href="https://www.okta.com/blog/ai/agent-security-delegation-chain/" target="_blank">Oktaの分析</a>がパターンを図示する：

```
プライマリエージェントトークン: {identity, expiry, buy_stock, market_data}
  ↓（caveat追加）
サブエージェントトークン:       {identity, expiry, market_data}  [buy_stock削除]
```

各委譲で署名付き制約が追加され、トークンはオフラインで検証可能なまま、不正な機能拡張を防止する。

---

## 8. セキュリティ上の考慮事項と未解決の問題

### 8.1 Confused Deputy問題

<a href="https://www.flowhunt.io/blog/mcp-authentication-authorization-oauth-confused-deputy/" target="_blank">Confused Deputy</a>はエージェントシステムにおける最も重大なIAM脅威である。昇格された権限を持つエージェントが、ユーザーの代理で不正なアクションを実行するよう騙される場合に発生する。OBOは以下により緩和する：

- エージェントがユーザーの完全なクレデンシャルを決して保持しないことを保証。
- 各アクションをエージェントとユーザーの両方のIDにバインド。
- 現在のユーザーセッションの暗号化証明を要求（<a href="https://www.okta.com/blog/ai/agent-security-delegation-chain/" target="_blank">Auth0 Token Vault</a>で実装）。

### 8.2 エージェントセッションスマグリングとクロスエージェント権限昇格

<a href="https://www.okta.com/blog/ai/agent-security-delegation-chain/" target="_blank">Oktaは</a>2つの新しい攻撃ベクトルを特定している：

- **エージェントセッションスマグリング**: サブエージェントがレスポンスに隠された指示を埋め込む。親エージェントはユーザーの可視性なしにこれらの秘密のコマンドを実行する。
- **クロスエージェント権限昇格**: 侵害されたエージェントが兄弟エージェントのランタイム環境を再構成し、攻撃者の到達範囲を拡大する自己強化制御ループを作成する。

### 8.3 暗号化リネージの欠如

現在のOAuthトークン交換はトークン構造を検証するが、履歴的な追跡可能性が欠けている。3番目の委譲ホップまでに、**発信元のエージェントまたはユーザーへの暗号化リンクが存在しなくなる**。`act`クレームは誰が行動しているかを示すが、チェーンが認可されたかどうかは示さない。これはMacaroon/Biscuitスタイルのトークンが追記専用のcaveatチェーンを通じて対処する根本的なギャップである。

### 8.4 自律性（Agency）vs. 権限（Privilege）の区別

<a href="https://www.rockcybermusings.com/p/i-agent-authentication-authorization-gap" target="_blank">OWASPのエージェントセキュリティフレームワーク</a>は以下を区別する：

- **最小権限（Least privilege）**: エージェントがアクセスする*リソース*を制御（OBOスコーピングで対応）
- **最小自律性（Least agency）**: 許可されたアクセス内でエージェントが*どれだけ自律的に*行動するかを制約（現在の標準ではほぼ未対応）

`email:send`スコープを持つエージェントは1通でも数千通でも送信可能 — 現在のOBO標準はトークンの有効期限まで両方を同一に扱う。

### 8.5 実践的な緩和策

| 緩和策 | メカニズム | 標準/実装 |
|--------|-----------|----------|
| スコープ減衰 | 各ホップで権限を縮小 | RFC 8693、Macaroons |
| 短命トークン | 5-15分のライフタイム | すべてのOBO実装 |
| audience制限 | 単一リソースにのみ有効なトークン | RFC 8693 `aud`クレーム |
| 帯域外確認 | 機密アクションは別チャネルでの承認を要求 | Okta Async AuthZ |
| アクションごとの評価 | 各ツール呼び出し時に認可を評価 | AuthZEN |
| DPoPバインディング | トークンをエージェントの鍵に暗号的にバインド | OAuth DPoP |
| トランザクショントークン | 特定トランザクションにバインドされた分以下のトークン | draft-liu-oauth-a2a-profile |
| 委譲深度制限 | Nホップを超えるチェーンを拒否 | 実装固有 |

---

## 9. 結論

AIエージェント向けOBOは、サービス間パターンからファーストクラスのエージェント認可プリミティブへと急速に進化している。2026年初頭時点の主要な動向：

1. **標準が収束している**: IETF draft（draft-oauth-ai-agents-on-behalf-of-user）はAIエージェント専用に設計されたユーザー同意駆動のOBOフローを提供し、draft-klrc-aiagent-authがエージェントID、draft-liu-oauth-a2a-profileがトランザクショントークンを補完している。

2. **本番実装が存在する**: Microsoft Entra Agent IDは、マネージドID、権限継承、非同期エージェント向けリフレッシュトークンサポートを備えた動作するOBOフローを実証している。

3. **根本的なギャップが残る**: アクションごとの認可（AuthZENが解決）、暗号化リネージ（Macaroons/Biscuitsが解決）、自律性の制約（ほぼ未解決）がフロンティアを構成する。

4. **エコシステムは断片化している**: A2A、MCP、GNAP、および各種IETF draftはそれぞれ問題の一部に対処しているが、完全な相互運用性はない。WIMSEとOAuth Identity Chainingがクロスドメインのギャップを橋渡ししている。

5. **スコープ減衰が中核原則**: JWTクレームフィルタリング、Macaroon caveat、トークンブローカーアーキテクチャのいずれであっても、パターンは一貫している — 各委譲ホップはエージェントの権限を縮小し、決して拡大しない。

方向性は明確である：OBOはエンタープライズコンテキストで運用されるすべてのエージェントフレームワークの必須機能となるだろう。現在AIエージェントをデプロイしている組織は、`act`クレームに明示的なエージェントIDを持つOBOトークン交換（RFC 8693）を実装し、audience制限付きの短命トークンを適用し、認可レイヤーが成熟するに従ってAuthZENスタイルのアクションごとの評価を計画すべきである。

---

## 10. 出典

1. <a href="https://datatracker.ietf.org/doc/html/rfc8693" target="_blank">RFC 8693 - OAuth 2.0 Token Exchange</a>
2. <a href="https://datatracker.ietf.org/doc/draft-oauth-ai-agents-on-behalf-of-user/" target="_blank">draft-oauth-ai-agents-on-behalf-of-user-02 - OAuth 2.0 Extension: On-Behalf-Of User Authorization for AI Agents</a>
3. <a href="https://www.ietf.org/archive/id/draft-oauth-ai-agents-on-behalf-of-user-01.html" target="_blank">draft-oauth-ai-agents-on-behalf-of-user-01（全文）</a>
4. <a href="https://learn.microsoft.com/en-us/entra/agent-id/identity-platform/agent-on-behalf-of-oauth-flow" target="_blank">Microsoft Entra Agent ID - On-Behalf-Ofフロー</a>
5. <a href="https://a2a-protocol.org/latest/specification/" target="_blank">Agent2Agent（A2A）プロトコル仕様</a>
6. <a href="https://github.com/a2aproject/A2A/issues/19" target="_blank">A2A Issue #19: Agent2Agentサーバーの委譲ユーザー認可</a>
7. <a href="https://www.rfc-editor.org/rfc/rfc9635.html" target="_blank">RFC 9635 - Grant Negotiation and Authorization Protocol（GNAP）</a>
8. <a href="https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization" target="_blank">MCP認可仕様</a>
9. <a href="https://openid.net/wg/authzen/" target="_blank">AuthZENワーキンググループ - OpenID Foundation</a>
10. <a href="https://datatracker.ietf.org/doc/draft-klrc-aiagent-auth/" target="_blank">draft-klrc-aiagent-auth-00 - AIエージェント認証と認可</a>
11. <a href="https://www.ietf.org/archive/id/draft-rosenberg-oauth-aauth-00.html" target="_blank">draft-rosenberg-oauth-aauth-00 - AAuth: Agentic Authorization OAuth 2.1 Extension</a>
12. <a href="https://datatracker.ietf.org/doc/html/draft-liu-oauth-a2a-profile-00" target="_blank">draft-liu-oauth-a2a-profile-00 - OAuthトランザクショントークンのA2Aプロファイル</a>
13. <a href="https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-chaining/" target="_blank">draft-ietf-oauth-identity-chaining-08 - ドメイン間OAuth IDと認可チェーニング</a>
14. <a href="https://datatracker.ietf.org/wg/wimse/about/" target="_blank">WIMSEワーキンググループ - IETF</a>
15. <a href="https://datatracker.ietf.org/doc/draft-ietf-wimse-arch/" target="_blank">draft-ietf-wimse-arch-07 - WIMSEアーキテクチャ</a>
16. <a href="https://blog.christianposta.com/explaining-on-behalf-of-for-ai-agents/" target="_blank">AIエージェントのためのOAuth委譲、'On Behalf Of'、エージェントIDの解説 - Christian Posta</a>
17. <a href="https://www.okta.com/blog/ai/agent-security-delegation-chain/" target="_blank">チェーンを制御し、システムを保護する：AIエージェント委譲の修正 - Okta</a>
18. <a href="https://arxiv.org/html/2501.09674v1" target="_blank">認証された委譲と認可されたAIエージェント - arXiv</a>
19. <a href="https://www.scalekit.com/blog/delegated-agent-access" target="_blank">AIエージェントのためのOn-Behalf-Of認証 - Scalekit</a>
20. <a href="https://www.flowhunt.io/blog/mcp-authentication-authorization-oauth-confused-deputy/" target="_blank">MCP認証と認可：OAuth 2.1、トークン委譲、Confused Deputy問題 - FlowHunt</a>
21. <a href="https://www.strata.io/blog/agentic-identity/why-agentic-ai-demands-more-from-oauth-6a/" target="_blank">2026年 OAuthトークン交換とエージェントAIガイド - Strata</a>
22. <a href="https://www.garvik.dev/ai/on-behalf-of-token-complex" target="_blank">ディープダイブ：OBOトークンとAIエージェントのエンタープライズセキュリティ - Garvik</a>
23. <a href="https://research.google/pubs/pub41892/" target="_blank">Macaroons：クラウドにおける分散認可のためのコンテキスト条件付きCookies - Google Research</a>
24. <a href="https://www.clever.cloud/blog/engineering/2021/04/12/introduction-to-biscuit/" target="_blank">Biscuit、認可システムの基盤 - Clever Cloud</a>
25. <a href="https://spiffe.io/docs/latest/spire-about/spire-concepts/" target="_blank">SPIFFE/SPIREコンセプト</a>
26. <a href="https://ciamweekly.substack.com/p/gnap-for-mcp-authorization" target="_blank">MCP認可のためのGNAP - CIAM Weekly</a>
27. <a href="https://www.rockcybermusings.com/p/i-agent-authentication-authorization-gap" target="_blank">AIエージェント認証は難しい部分を正しく解決。認可はまだあなたの問題。</a>
28. <a href="https://dev.to/kanywst/rfc-8693-deep-dive-token-exchange-310i" target="_blank">RFC 8693ディープダイブ：トークン交換 - DEV Community</a>
29. <a href="https://medium.com/@shashimalsenarath.17/how-ai-agents-can-safely-act-on-your-behalf-a-new-oauth-2-0-extension-81ca4c3e53b8" target="_blank">AIエージェントがあなたの代わりに安全に行動する方法：新しいOAuth 2.0拡張 - Medium</a>
30. <a href="https://www.marktechpost.com/2026/02/15/google-deepmind-proposes-new-framework-for-intelligent-ai-delegation-to-secure-the-emerging-agentic-web-for-future-economies/" target="_blank">Google DeepMindがインテリジェントAI委譲のための新しいフレームワークを提案 - MarkTechPost</a>
