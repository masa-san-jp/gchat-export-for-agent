/**
 * Google Chat → Drive JSON 差分同期スクリプト
 *
 * 事前準備：
 * 1. GCPプロジェクトでChat APIを有効化
 * 2. OAuthスコープに https://www.googleapis.com/auth/chat.messages.readonly を追加
 * 3. SPACE_IDS に対象スペースのIDを列挙
 * 4. DRIVE_FOLDER_ID に保存先DriveフォルダIDを設定
 * 5. トリガーで syncAllSpaces を定期実行（5〜60分おき）
 */

const SPACE_IDS = [
  'spaces/XXXXXXX', // スペースID1
  'spaces/YYYYYYY', // スペースID2
];

const DRIVE_FOLDER_ID = 'YOUR_FOLDER_ID';

// ---------------------------------------------------
// メイン：全スペースを同期
// ---------------------------------------------------
function syncAllSpaces() {
  SPACE_IDS.forEach(spaceId => syncSpace(spaceId));
}

// ---------------------------------------------------
// スペース単体の差分同期
// ---------------------------------------------------
function syncSpace(spaceId) {
  const props = PropertiesService.getScriptProperties();
  const propKey = `lastSync_${spaceId.replace('/', '_')}`;
  const lastSync = props.getProperty(propKey) || '1970-01-01T00:00:00Z';

  const newMessages = fetchMessages(spaceId, lastSync);
  if (newMessages.length === 0) return;

  const fileName = `${spaceId.replace('spaces/', 'chat_')}.json`;
  const folder = DriveApp.getFolderById(DRIVE_FOLDER_ID);

  // 既存ファイルの読み込み
  let existing = [];
  const files = folder.getFilesByName(fileName);
  if (files.hasNext()) {
    const file = files.next();
    try {
      existing = JSON.parse(file.getBlob().getDataAsString());
    } catch (e) {
      existing = [];
    }
    // 差分をマージして上書き
    const merged = mergeMessages(existing, newMessages);
    file.setContent(JSON.stringify(merged, null, 2));
  } else {
    // 新規作成
    folder.createFile(fileName, JSON.stringify(newMessages, null, 2), MimeType.PLAIN_TEXT);
  }

  // タイムスタンプを更新
  props.setProperty(propKey, new Date().toISOString());
  Logger.log(`${spaceId}: ${newMessages.length}件追加`);
}

// ---------------------------------------------------
// Chat API からメッセージ取得（ページネーション対応）
// ---------------------------------------------------
function fetchMessages(spaceId, since) {
  const messages = [];
  let pageToken = null;

  do {
    const params = {
      pageSize: 1000,
      filter: `createTime > "${since}"`,
    };
    if (pageToken) params.pageToken = pageToken;

    const query = Object.entries(params)
      .map(([k, v]) => `${k}=${encodeURIComponent(v)}`)
      .join('&');

    const url = `https://chat.googleapis.com/v1/${spaceId}/messages?${query}`;
    const res = UrlFetchApp.fetch(url, {
      headers: { Authorization: `Bearer ${ScriptApp.getOAuthToken()}` },
      muteHttpExceptions: true,
    });

    const data = JSON.parse(res.getContentText());
    if (data.messages) messages.push(...data.messages.map(normalizeMessage));
    pageToken = data.nextPageToken || null;

  } while (pageToken);

  return messages;
}

// ---------------------------------------------------
// メッセージを必要な項目に絞って正規化
// ---------------------------------------------------
function normalizeMessage(msg) {
  return {
    id: msg.name,
    sender: msg.sender?.displayName || '',
    text: msg.text || '',
    createTime: msg.createTime,
    thread: msg.thread?.name || '',
  };
}

// ---------------------------------------------------
// IDベースで重複除去しながらマージ
// ---------------------------------------------------
function mergeMessages(existing, newMessages) {
  const map = new Map(existing.map(m => [m.id, m]));
  newMessages.forEach(m => map.set(m.id, m));
  return Array.from(map.values()).sort((a, b) =>
    new Date(a.createTime) - new Date(b.createTime)
  );
}
