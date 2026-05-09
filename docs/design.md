# Google Chat 履歴同期システム 設計仕様書

## 1. 概要

Google Chat の各スペースのメッセージ履歴を定期的に取得し、Google Drive 上の JSON ファイルへ差分反映する。Drive for Desktop 経由でローカルにも自動同期し、AIエージェントがローカル・クラウド双方から参照できる状態を維持する。

---

## 2. システム構成

```
Google Chat
    │
    │ Chat REST API（差分取得）
    ▼
Google Apps Script（定期実行）
    │
    │ DriveApp.setContent()
    ▼
Google Drive（JSON ファイル）
    │
    │ Drive for Desktop（自動同期）
    ▼
ローカルファイルシステム
    │
    ├─ Claude Code などローカルエージェント
    └─ Claude.ai（Drive MCP経由でも参照可）
```

---

## 3. コンポーネント定義

### 3.1 GAS スクリプト

| 項目 | 内容 |
|------|------|
| 実行環境 | Google Apps Script（V8ランタイム） |
| トリガー | 時間主導型トリガー（5〜60分間隔、要件に応じて設定） |
| エントリポイント | `syncAllSpaces()` |
| 対象スペース | `SPACE_IDS` 配列に列挙したスペースを順次処理 |

### 3.2 状態管理

| 項目 | 内容 |
|------|------|
| 使用サービス | `PropertiesService.getScriptProperties()` |
| キー命名規則 | `lastSync_{spaceId}` （例: `lastSync_spaces_XXXXXXX`） |
| 保存値 | 前回正常終了時の ISO 8601 タイムスタンプ |
| 初期値 | `1970-01-01T00:00:00Z`（未実行時は全履歴取得） |

### 3.3 Drive ファイル

| 項目 | 内容 |
|------|------|
| 保存先 | `DRIVE_FOLDER_ID` で指定したフォルダ |
| ファイル名規則 | `chat_{spaceId}.json`（例: `chat_XXXXXXX.json`） |
| フォーマット | JSON（インデント2スペース） |
| MIMEタイプ | `MimeType.PLAIN_TEXT`（拡張子 `.json`） |

---

## 4. データフロー

### 4.1 差分取得フロー

```
1. PropertiesService から lastSync タイムスタンプを取得
2. Chat API に filter="createTime > {lastSync}" でリクエスト
3. ページネーション（nextPageToken）が尽きるまで繰り返し取得
4. 全件取得後、PropertiesService の lastSync を現在時刻で更新
```

### 4.2 マージフロー

```
1. Drive の既存 JSON ファイルを読み込み（存在しない場合は空配列）
2. 既存データを Map<id, message> に変換
3. 新規メッセージを Map に上書き追加（重複IDは上書き）
4. Map を配列化し createTime 昇順でソート
5. Drive ファイルに上書き保存
```

---

## 5. データスキーマ

### 5.1 保存 JSON 構造

```json
[
  {
    "id": "spaces/XXXXXXX/messages/YYYYYYY",
    "sender": "田中太郎",
    "text": "メッセージ本文",
    "createTime": "2026-05-09T10:00:00.000Z",
    "thread": "spaces/XXXXXXX/threads/ZZZZZZZ"
  }
]
```

### 5.2 フィールド定義

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `id` | string | メッセージのリソース名（一意キー） |
| `sender` | string | 送信者の表示名 |
| `text` | string | メッセージ本文（プレーンテキスト） |
| `createTime` | string | メッセージ作成日時（ISO 8601） |
| `thread` | string | 所属スレッドのリソース名 |

---

## 6. 設定パラメータ

| 定数 | 説明 | 設定箇所 |
|------|------|---------|
| `SPACE_IDS` | 対象スペースIDの配列 | スクリプト冒頭 |
| `DRIVE_FOLDER_ID` | 保存先DriveフォルダID | スクリプト冒頭 |
| トリガー間隔 | 5〜60分（任意） | GASトリガー設定画面 |

---

## 7. 必要な権限・事前設定

| 項目 | 内容 |
|------|------|
| GCP API | Chat API を有効化 |
| OAuthスコープ | `https://www.googleapis.com/auth/chat.messages.readonly` |
| GASサービス | 「サービスの追加」から Google Chat API を追加 |
| Drive for Desktop | ローカル同期が必要な場合のみインストール |

---

## 8. 制約・注意事項

| 項目 | 内容 |
|------|------|
| リアルタイム性 | トリガー間隔に依存。厳密なリアルタイムは不可 |
| スペース単位 | 全スペース一括取得は不可。`SPACE_IDS` に明示的に列挙が必要 |
| Chat API 制限 | 1ページあたり最大1000件。大量履歴の初回取得は複数ページにわたる |
| GAS 実行時間制限 | 1回の実行は最大6分。スペース数が多い場合は分割実行を検討 |
| 添付ファイル | 本仕様では添付ファイル・リアクションは取得対象外 |
| ローカル同期ラグ | Drive for Desktop の同期は数秒〜最大1分程度の遅延あり |

---

## 9. 拡張ポイント

- **添付ファイル対応**：`msg.attachment` フィールドを `normalizeMessage` に追加
- **リアクション取得**：Chat API の `reactions.list` エンドポイントを追加
- **Markdown変換**：JSON蓄積後、別GAS関数でMD形式に変換して二重保存
- **スペース自動検出**：`spaces.list` APIで参加スペースを自動列挙し `SPACE_IDS` を動的生成
