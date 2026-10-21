# 高瀬さんと話す

由衣さんがブラウザのマイクで話し、高瀬さんが声で返す静的な音声チャットです。GitHub Pages に置きます。

公開先の想定 URL は [https://ayumisan.github.io/slot-060/](https://ayumisan.github.io/slot-060/) です。サイト本体は `docs/` です。

## 使い方

1. ページを開く。
2. 「設定」に ElevenLabs の API キーを入れる。
3. 「このブラウザに保存」を押す。キーは `localStorage`（`slot060.elevenlabsApiKey`）にだけ残る。
4. 「接続する」を押して、マイクを許可する。
5. 話す。こちらが話し終えると高瀬さんが返す。もう一度「切断する」で終了。
6. 使い終わったら「キーを削除」。

API キーだけ入れれば話せます。声の欄は空でも、接続時に既定の voice id がセッションへ送られます。モデルの既定は `eleven_realtime_v1_mini` で、設定の Mini / Max で切り替えます。温度は 0.70、反応の早さは high、割り込みはオンが既定です。これらもこのブラウザの `localStorage` に保存されます。

speed、stability、similarity_boost、style のスライダーはありません。Realtime はそれらを黙って無視し、`session.speed` は `unknown_parameter` でエラーになります。数値は `docs/config.js` の `VOICE_SETTINGS_NOT_SENT` に残してあり、送信しません。ハキハキしたテンポは `docs/persona.js` の「ハキハキしたテンポの良い口調」で出します。

## 声

既定の voice id は [`docs/config.js`](docs/config.js) の `DEFAULT_VOICE_ID` だけです。高瀬さんは男性で、いまの声は落ち着いた男性の Yusuke（`94gBdmmazJ025HbvF78b`）です。高瀬専用の声ができたら、この定数か設定欄の id を差し替えてください。

セッション発行と、接続直後の `session.update` は同じ形です。声は `session.audio.output.voice` にだけ入れます。`session.voice` は `unknown_parameter` で拒否されます。値はこのワークスペースにある voice id です。Yusuke（`94gBdmmazJ025HbvF78b`）を既定にしています。`jessica` や `lily`、OpenAI の声名のような既定名も使えます。入力音声は `session.audio.input.format` の `audio/pcm`、24000 Hz です。

差し替え方はどちらかです。

1. ページの「声の ID」に新しい id を書いて「このブラウザに保存」する。保存値は `localStorage` の `slot060.voice` で、このブラウザだけが変わります。空欄にして保存すると `DEFAULT_VOICE_ID` に戻ります。
2. 全員の既定を変えるときは `docs/config.js` の `DEFAULT_VOICE_ID` を高瀬の voice id に書き換えて再公開する。すでに設定欄へ別の id を保存したブラウザは、その保存値が優先されます。戻すときは声の欄を空にして保存してください。

## API キー

キーはリポジトリに含まれていません。ページにも最初からは入っていません。

- 保存先はこのブラウザの `localStorage` だけです。
- 「キーを削除」でその項目を消します。
- この Pages は `https://ayumisan.github.io` を他のプロジェクトサイトと共有します。`localStorage` はパスごとではなくオリジンごとなので、同じオリジンの別ページからも読める可能性があります。共有パソコンでは削除してください。
- 接続時、キーは短命の client secret を発行するためだけに `POST https://api.elevenlabs.io/v1/convai/realtime/client_secrets` へ送られます。WebSocket には返ってきた `value` だけを使います。

2026-10-20 時点で、この発行エンドポイントは `Access-Control-Allow-Origin: *` と `Access-Control-Allow-Headers: *` を返すため、静的ページのブラウザから発行できます。公式ドキュメントは、キーをブラウザに置かずサーバーで発行する流れを推奨しています。ブラウザからの発行が CORS や組織の方針で止まった場合は、下の任意サーバーを使ってください。

## 接続

- モデル: 設定の Mini / Max。既定は `eleven_realtime_v1_mini`。WebSocket の `model` クエリとセッションの `model` は同じ値
- WebSocket: `wss://api.elevenlabs.io/v1/convai/realtime?model=<選択したモデル>`
- ブラウザ認証: サブプロトコル `realtime` と `openai-insecure-api-key.<client secret>`（OpenAI Realtime SDK と同じ形）
- 入力: `session.audio.input.format` は `{ "type": "audio/pcm", "rate": 24000 }`。送るバイトはモノラル PCM16 リトルエンディアンで、WAV ヘッダは付けない
- 文字起こし: `session.audio.input.transcription.language` は常に `ja`
- 区切り: `turn_detection.type` は `semantic_vad`。`server_vad` はエラーになる。`eagerness` は low / medium / high（既定 high）。`interrupt_response` は設定の割り込みトグル
- 温度: `temperature`。スライダーは 0.30 から 1.20、既定 0.70
- 長さ: `max_duration_seconds` は 600。`max_response_output_tokens` は受け取れるが、スライダーは置いていない
- 声: `session.audio.output.voice`。`session.voice` は送らない
- 送らない: `stability`、`similarity_boost`、`style`、`speed`、`voice_settings`。`session.speed` は `unknown_parameter`
- 送るイベント: `input_audio_buffer.append`
- 受ける音声: `response.output_audio.delta`（古い `response.audio.delta` も再生）
- 文字起こしイベント: 入力は `conversation.item.input_audio_transcription.completed`、返事は `response.output_audio_transcript.delta`
- 割り込み: `input_audio_buffer.speech_started` で再生を止める
- 発話中は立ち絵がわずかに揺れます

サーバーが `Realtime models are not available for this workspace` を返したときは、その文字列をそのまま画面上部のエラー欄に出します。他の API エラーも同じ欄です。

## 任意のトークンサーバー

ブラウザから client secret を発行できないときだけ使います。

```bash
ELEVENLABS_API_KEY="あなたのキー" node optional-token-server/server.mjs
```

ページの「トークン取得 URL」に `http://127.0.0.1:8787/token` を入れて保存すると、ページは API キーの代わりにその URL へ GET し、応答の `value` で WebSocket を開きます。キーはサーバープロセスの環境変数にだけ置いてください。このプロセスを公開インターネットに出さないでください。誰でも届く状態だと、届いた人がトークンを発行できます。

## 人物設定

高瀬さんのシステムプロンプトは編集用に [`docs/persona.js`](docs/persona.js) へ分けてあります。

`https://ayumisan.github.io/slot-096` は GitHub Pages の "Site not found"（HTTP 404）、`https://github.com/ayumisan/slot-096` は Repository not found（HTTP 404）でした。ページもリポジトリも読めなかったので、そこからの原文転記はできていません。プロンプトに入れた事実とセリフは、同じ高瀬航・小野寺由衣の公開作品 [ayumisan/slot-097](https://github.com/ayumisan/slot-097)（『店じまいの合図』）にあるものだけです。呼び方の「由衣さん」は、このアプリで相手をそう呼ぶ、という指定です。作品中の呼称は「小野寺さん」から「由衣さん」です。

使った公開上の要点:

- 題名は『店じまいの合図』。高瀬 航（たかせ わたる、32）と表田 由衣（29）
- 場所は雑居ビル3階のプライベートサロン。トレーニングのあと、奥のベッドで整体もする
- 入会金・月額の縛りはなく、都度払い。連絡は LINE
- 高瀬は男性。身長178cm、体重78kg。がっしりした筋肉質で、細マッチョではない。肩幅がめっちゃ広い。口調はハキハキしたテンポの良いトレーナーで、がっしりした体に合う安心感はある
- 由衣さんは身長160cm。差は18cmで、高瀬は見下ろし、話しかけるときは少し身をかがめる
- 高瀬はチャコールの施術用ポロ、右頬の浅いえくぼ。一人称は「俺」
- 由衣さんの設定は、29歳・160cm・会社員だけにしぼっている
- 呼び方: 高瀬さんは自分を「高瀬航です」と名乗り、由衣さんからは「高瀬さん」と呼ばれる。読みは「わたる」で、「こう」とは読まない
- 高瀬さんは由衣さんを、ふだんは「由衣さん」と呼ぶ。距離が縮まったら「由衣」と呼び捨てにして、少しじらすように呼ぶ（軽く、やりすぎない）
- 小野寺さんは由衣さんとは別の人で、「小野寺さん」と呼ぶ
- 筋トレ中、いいフォームのときはボディビルの掛け声で褒める（例:「肩にでっかい重機乗せてんのかい！」「キレてる！」）。ここぞのときに一言だけ
- 軽い口調で、脈やフォームには触れる。心のうちを断定せず、「そういうことにしておきましょうか」と置く
- 店を終える合図は「今日はこれで店じまいだから」

slot-096 が後から公開されたら、`docs/persona.js` の文字列だけ差し替えてください。

## 画像

`docs/assets/` は [slot-097 の assets](https://github.com/ayumisan/slot-097/tree/main/assets) からコピーしています。パスは `assets/...` の相対パスです。

| ファイル | 画面での役割 |
| --- | --- |
| `assets/takase_main.png` | 高瀬さんの立ち絵。チャットの横（狭い画面では上） |
| `assets/bg_entrance.png` | ページ全体の背景。サロンの玄関 |
| `assets/bg_training.png` | 右側の小さな装飾。「フロア」 |
| `assets/expressions/takase/02_笑顔.png` | 見出し横の印章と、高瀬さんの発言アイコン |
| `assets/expressions/takase/07_楽しみ.png` | 見出し横の印章。話しているあいだ光る |
| `assets/expressions/yui/training_expression_02.png` | 「由衣さん」の装飾と、利用者側の発言アイコン |

## GitHub Pages

ワークフローは `.github/workflows/pages.yml` です。`main` への push で `docs/` を Pages に出します。

プライベートリポジトリの Pages は、無料プランでは有効にできないことがあります。その場合は GitHub の Settings、Pages で Source を GitHub Actions にしてください。ブランチから出す場合は `main` の `/docs` です。

期待する URL: https://ayumisan.github.io/slot-060/

## 開発用チェック

```bash
node tests/audio-util.test.mjs
node --check docs/app.js
node --check optional-token-server/server.mjs
```

マイクと実 API への発話は、有効なキーを入れたブラウザで確認してください。
