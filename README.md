# fizz-x-mention-source

**Fizz** コメント入力系部品 — X (旧 Twitter) のメンションを XAPI proxy 経由で
polling し、正規化コメント ([fizz-protocol](https://github.com/aiviecast/fizz-protocol)
の `Comment`、source = `XMention`) を NDJSON で stdout に流す。

責務は一行: **XAPI /api/mentions → comment ストリーム**。
投稿 (tweet) はしない — それは x-control 系部品の責務。
dedupe は [fizz-comment-dedupe](https://github.com/aiviecast/fizz-comment-dedupe) の責務
(sinceId 前進により通常は重複しないが、保険として通す)。

## 前提: XAPI proxy

X API v2 を直接叩かず、OAuth 2.0 user context + token refresh を巻き取る
ローカル **XAPI** サーバ (別リポジトリ) 経由で読む。認証は XAPI が発行する
API key の Bearer。

想定エンドポイント (openaituber `server/x/realClient.ts` の計画仕様):

```
GET {XAPI_BASE_URL}/api/mentions?xUserId=...&sinceId=...
→ {"mentions": [{"id", "text", "createdAt",
                  "author": {"id", "handle", "displayName"}}, ...]}
```

> **注意**: 2026-06 時点で XAPI 側の `/api/mentions` は未実装
> (openaituber の realClient は空配列を返すスタブ)。本部品はこの計画仕様を
> ターゲットに実装している。XAPI 側の実装が異なる形になった場合は
> `parse_mentions` を合わせること。

## Usage

```bash
almide build src/main.almd -o fizz-x-mention-source

XAPI_BASE_URL=http://localhost:3000 XAPI_KEY=... XAPI_X_USER_ID=... \
  ./fizz-x-mention-source | fizz-comment-dedupe | ...
```

## 挙動

- 初回フェッチは起動前から溜まっていたメンション — 流さずに sinceId だけ
  前進 (priming)
- sinceId は前進のみ (空ページで巻き戻さない)。snowflake ID 比較は
  「桁数が違えば長い方が大きい、同じなら辞書順」
- `displayName` 空のときは `@handle` に倒す

## Env

| var | 意味 | default |
|---|---|---|
| `XAPI_BASE_URL` | XAPI のベース URL | (必須) |
| `XAPI_KEY` | XAPI 発行の API key (Bearer) | (必須) |
| `XAPI_X_USER_ID` | 対象アカウントの xUserId | (必須) |
| `X_MENTION_POLL_MS` | polling 間隔 | 60000 |

## Tests

```bash
almide test
```
