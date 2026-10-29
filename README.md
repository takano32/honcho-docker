# honcho-docker

[plastic-labs/honcho](https://github.com/plastic-labs/honcho)（エージェント用の記憶サーバー）を、この箱の共有サービスに繋いで動かす置き場。

- 本家は `./honcho` に clone して build する（このリポには含めない＝`.gitignore`）。いま使っているコミットは `96806bc29236331ed3f1749589b7b1442c2325e5`（2026-10-28）。ライセンスは AGPL-3.0。本家は改変しない。
- コンテナは `api`（起動時に alembic のマイグレーションを流してから fastapi。ホスト側は 127.0.0.1:18000 だけ）と `deriver`（メッセージから観察を抜き出すワーカー）の2つ。本家例の redis（キャッシュ、既定で無効）と mcp は入れていない。
- DB は [postgresql-docker](https://github.com/takano32/postgresql-docker) の共有 PostgreSQL（external network `postgresql`）の `honcho` データベース。
- 埋め込みは [ollama-docker](https://github.com/takano32/ollama-docker) の共有 ollama（external network `ollama`）の `bge-m3`（1024次元）。Honcho には埋め込みをコンテナ内で回す選択肢が無く、OpenAI 互換の口が要る。
- LLM は OpenCode Go の `deepseek-v4.1-flash` に直結（LiteLLM などの中継は挟まない）。9つのモジュール（deriver、summary、dream の2、dialectic の5段）すべてに同じ設定を置く。

## OpenCode Go のヘッダ

OpenCode Go は外のクライアントに、自分の名前の User-Agent と、会話ごとに固定の `x-opencode-session` を求めている（[OpenCode Go の文書](https://opencode.ai/v2/docs/console/go/)）。
本家の `ModelOverrideSettings.provider_params.extra_headers`（`src/config.py`）が OpenAI SDK の呼び出しにそのまま載るので、各モジュールの `..._OVERRIDES__PROVIDER_PARAMS` に JSON で渡している（compose の YAML アンカー `llm-params`）。
`x-opencode-session` は固定値で、会話ごとには変えられない（呼び出し側にセッション ID を渡す経路が無い）。

`patches/0001-openai-default-headers.patch` は、同じことを全クライアントの既定ヘッダとして足す試作。**使っていない**（本家に手を入れると更新のたびに当て直しになるため）。

## 初回の立ち上げ

1. `.env.example` を `.env` に写して `chmod 600`。`HONCHO_DB_PASSWORD` と `OPENCODE_API_KEY` を埋める。
2. 共有 PostgreSQL にロールと DB と拡張を作る（パスワードは `.env` と同じもの）。

   ```
   docker exec -i postgresql psql -U postgres \
     -c "CREATE ROLE honcho LOGIN PASSWORD '...';" -c "CREATE DATABASE honcho OWNER honcho;"
   docker exec -i postgresql psql -U postgres -d honcho \
     -c "CREATE EXTENSION IF NOT EXISTS vector;" -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;" -c "CREATE EXTENSION IF NOT EXISTS btree_gin;"
   ```

3. `git clone https://github.com/plastic-labs/honcho` して `docker compose build`。
4. **埋め込みの次元を合わせる**。マイグレーションは `vector(1536)` で列を作るので、1024 のまま起動すると api が `public.documents.embedding dim (1536) does not match EMBEDDING_VECTOR_DIMENSIONS (1024)` で止まる。空のうちに1回だけ流す。

   ```
   docker compose up -d api   # マイグレーションを流させる（ここで起動に失敗するのは想定どおり）
   docker compose stop api
   docker compose run --rm --no-deps --entrypoint /app/.venv/bin/python api scripts/configure_embeddings.py --yes
   docker compose up -d
   ```

## 確かめたこと（2026-10-29）

- api が healthy、`/health` が `{"status":"ok"}`。
- 使い捨ての workspace に日本語のメッセージを2件入れ、deriver が DeepSeek flash で観察3件を作った。検索と dialectic（`/peers/{id}/chat`、minimal）も答えた。
- deriver は既定でメッセージをためてから処理する（`DERIVER_REPRESENTATION_BATCH_TARGET_INPUT_TOKENS`、最長 `REPRESENTATION_BATCH_MAX_AGE_SECONDS`=1800 秒）。試験のときだけ `DERIVER_FLUSH_ENABLED=true` で即時に流した。
- 観察は英語で書かれる（入力が日本語でも）。

## まだ決めていないこと

- 認証（`HONCHO_AUTH_USE_AUTH`）。いまは false で、127.0.0.1 にしか開けていない。

## 位置づけ（2026-10-29 決定）

nilkani06 の記憶の本体は Hindsight のまま。Honcho のフックへの切り替えはしない。
理由は、全文検索が `to_tsvector("english", …)`（`src/utils/search.py`）で日本語が単語に割れないこと、観察が英語で書かれること、チャンネル分離にあたる印の仕組みが無いこと。
Honcho は立てたまま、Claude Code からは MCP（`@honcho-ai/claude-honcho` の mcp-server.js を bun の自動取得で起こす）で様子を見る。workspace は `nilkani06` と `zerozari00` を作ってある（中身は空）。

## つまずき

- workspace を消した直後に同じ名前で作り直すと、api が消す前の workspace をプロセス内に覚えていて、作ったふりをして古い uid を返す（DB には入らない。`CACHE_ENABLED=false` でもプロセス内のキャッシュは残る）。`docker compose restart api` してから作り直せば入る（2026-10-29）。消すときは、先に session を全部消さないと workspace の削除が 409 で断られる。
