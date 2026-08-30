# KnowLocal

[日本語](./README.md) | [English](./README.en.md)

社内のPDF・Word・Excel・PowerPointなどを取り込み、ローカルLLMで質問に答えるRAG（Retrieval-Augmented Generation）基盤です。
資料本文は外部クラウドへ一切送信せず、検索から回答まで社内LANだけで完結します。

> **本リポジトリで公開しているのはプログラム本体のみです。**
> サンプルデータ・参照資料等は含まれていません。ご自身の文書を投入してご利用ください。

詳細: https://www.aiadvocate.jp/knowlocal

## 特徴

- 意味検索（pgvector）＋キーワード検索（BM25）をRRFで統合するハイブリッド検索
- BGE-reranker-v2-m3によるリランク、MMRによる根拠選択（重複除去・多様性確保）
- 複数のローカルLLM（Ollama: qwen3 / gemma3 等）に同じ根拠で回答させ、根拠整合性スコア最高のものを自動採用
- 監視ログ（ai-monitor-agent3）とも同じ基盤で横断検索が可能
- 検索精度を数値（MRR）で評価しながら育てられる設計
- 実測例（743command.pdf・コマンド集・17問）: ベクトル検索のみ MRR 0.146 → ハイブリッド＋リランクで **0.618**
  ※数値は資料・質問セットにより変動します。導入時は自部門の評価セットで測定してください。

## 対応形式

PDF / Word / Excel / PowerPoint / CSV / Markdown / TXT

## 全体フロー

```
PDF/Word/Excel/PPT/CSV/MD/TXT
        ↓
  doc_embedding.py（テキスト抽出・分割）
        ↓
PostgreSQL+pgvector   figure_rag/ColPali（図・視覚索引・任意）
        ↓
     ユーザーの質問
        ↓
意味検索(pgvector)  ×  キーワード(BM25)
        ↓
      RRF ハイブリッド
        ↓
   リランク（BGE-reranker-v2-m3）
        ↓
      MMR で根拠選択
        ↓
   Ollama（qwen3 / gemma3）
        ↓
  根拠付き回答 → Open WebUI
```

※ 外部クラウドへ資料・質問は送信しません（モデルの初回ダウンロードを除く）。

## 動作要件

推奨（現行の本番運用相当）と最小構成の例です。実務ではNVIDIA GPUを前提としています。

| 項目 | 推奨 | 最小・代替 |
|---|---|---|
| OS | Linux / Windows（両対応） | どちらでも可 |
| GPU | NVIDIA GPU 推奨 | CPUのみは可だが実用上は低速 |
| VRAM目安 | 16GB（図表・best-of利用時） | 8GB（小モデル） |
| LLM | Ollama（GPU） | LM Studioも可 |
| DB | PostgreSQL + pgvector | Docker またはネイティブ |
| UI | Open WebUI（任意） | CLIのみでも可 |
| 監視 | ai-monitor-agent3（任意） | 文書RAGだけでも可 |
| Python | 3.12 | — |

インターネット公開は想定していません（社内LAN向け）。

## セットアップ

### Linux

```bash
git clone https://github.com/ai-advocate-jp/knowlocal.git
cd knowlocal

# pgvector（Docker）
docker compose up -d

# Python環境
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# ローカルLLMモデルの取得
ollama pull qwen3:8b gemma3:12b

# サーバー起動
uvicorn best_of_server:app --host 0.0.0.0 --port 8100
```

### Windows（Dockerなしでも可）

- PostgreSQL + pgvector をネイティブ導入
- Ollama for Windows + 最新NVIDIAドライバを導入
- CUDA版 torch を先に入れてから `pip install -r requirements.txt`

詳細は `MANUAL.md` 第8章を参照してください。

## 文書の取り込み・埋め込み

1. `doc_embedding.py` で対象文書（PDF / Word / Excel / PowerPoint / CSV / Markdown / TXT）を読み込み、テキスト抽出・分割します。
   - チャンク分割: `chunk_size=500` / `overlap=100`
   - 区切りは 空行 → 改行 → 。 → 、 の優先順で、文の途中では切りません。境界で文が切れても前後100字を重ね、情報欠損を防ぎます。
2. 図表は後から `--figures-only` オプションで追加でき、既存のテキスト索引を壊しません。
3. 埋め込みモデル: `paraphrase-multilingual-MiniLM-L12-v2`（日本語対応）
   - 社内文書は PostgreSQL + pgvector、監視ログは FAISS + pickle と使い分けています。
4. 取り込み後はAPIの再起動不要（ホットリロード）です。

```bash
python doc_embedding.py --source ./documents
```

## 検索・回答の仕組み

1. 意味検索（pgvector）とキーワード検索（BM25）を RRF で統合
2. BGE-reranker-v2-m3（Cross-Encoder）でリランク
3. MMR で重複を除いて多様な根拠を選択
4. Ollama にpullした複数モデル（例: `qwen3:8b`, `gemma3:12b`）に同じ根拠で回答させ、根拠整合性スコア最高のものを自動採用
5. 採用モデル名・スコア・出典（資料・ページ）を回答末尾に表示

CLIで `--show-all` を指定すると全モデルの回答を確認できます（モデル数が増えるほど処理時間も増加します）。

補足（検索精度の実測から）:
- コマンド名・型番が多い資料ではBM25が効きます
- リランクはハイブリッド検索と組み合わせて初めて効果を発揮します
- 「ベクトル検索だけ入れれば精度が上がる」という想定は誤りです

## 権限・運用

- ロール（viewer / staff / admin）、文書ACL、監査ログは実装済みです
- 既定値は `auth_required=false` です。部署横断の運用設計・SSO本格導入は今後の強化対象です

## 制約

- 実務はNVIDIA GPU前提（CPUのみでも動作しますが低速です）
- 16GB VRAM環境では、図検索と画像生成の同時実行に注意してください
- 似たコマンドが並ぶマニュアルでは別ページを回答してしまうことがあります
  → 回避策: 出典を複数表示する／「p.○の手順は？」とページ指定する／章ごとに分割して取り込む

## 他ツールとの位置づけ

Dify・LangChain・LM Studioとは競合ではなく、レイヤーが異なります。KnowLocalが強いのは検索精度・説明可能性・ローカル完結性で、非エンジニア向けUIやAgent機能・コネクタの幅は弱みです。ここで培った精度ノウハウは、Dify/LangChainへ載せ替えても活かせます。

## ライセンス

本プロジェクトは [MIT License](./LICENSE) のもとで公開されています。

## 免責事項

本ツールは現状有姿（as-is）で提供されます。投入した文書の取り扱いについては利用者の責任において行ってください。

## 関連プロジェクト

- [AI Monitor Agent](https://www.aiadvocate.jp)（GPU・システム監視エージェント、Rust製）

## 開発

Across Systems Corporation / [AI Advocate](https://www.aiadvocate.jp)
