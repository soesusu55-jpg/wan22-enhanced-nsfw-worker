# Wan2.2 Enhanced NSFW — RunPod Serverless デプロイキット

gfaione/wan2.2_enhanced_nsfw（FAST MOVE V2 Q4_K_M GGUF）を、RunPod Serverless + ComfyUI で動かすためのテンプレートです。
Remix 14Bキット（`../wan22-remix-14b-runpod/`）から派生した別エンドポイント用で、Remix側はそのまま残します。

## なぜ Enhanced NSFW か

- Lightningがモデルに統合済み（モデルページ: "All models already include the Lightning LoRAs"）。LightX2V LoRAを追加で載せないこと。画質が劣化する。
- 推奨設定は **ステップ 2+2、euler / simple、CFG 1**。今のRemix＋LightX2Vと同じ4ステップだが、モデルファイルが小さい（Q4_K_Mペアで約19.3GB、Remix fp8ペアは約28.6GB）のでロードが軽い。
- GGUFのため `city96/ComfyUI-GGUF`（UnetLoaderGGUF）が必要。モデルは `models/unet/` に置く。
- 5分以内を保証する実測はない。Remixと同じくL40Sで161フレームが5分に収まるかはテストで確認する。

## 10秒・開始画像＋終了画像（FLF2V）

デフォルトの `api-workflow.json` は **FLF2V（First-Last Frame to Video）** です。
Remixキットと同じ構成で、UNETLoaderをUnetLoaderGGUFに置き換え、LoRAノードを外しただけです。

- 解像度：720×1280（縦長）
- フレーム数：**161フレーム**（16fps × 10秒 + 1。Wanは 4n+1 フレーム）
- fps：16（`VHS_VideoCombine` の frame_rate も16）
- サンプリング：総ステップ4（high 2 / low 2）、euler / simple、cfg 1、shift 5
- 呼び出し：`start_image_b64`（開始画像）と `end_image_b64`（終了画像）を base64で渡す。`image_b64` は開始画像の別名として使える。プロンプトは `prompt` / `negative_prompt` で上書きできる。
- 実行タイムアウト：デフォルト **1800秒**。テスト時は5分ルールでキャンセルすること。

## 必要なファイル（4つ）

| ファイル | 置き場所 | サイズ目安 |
|---|---|---|
| `wan22EnhancedNSFWSVICamera_nsfwFASTMOVEV2Q4KMH.gguf` | `ComfyUI/models/unet/` | 9.65GB |
| `wan22EnhancedNSFWSVICamera_nsfwFASTMOVEV2Q4KML.gguf` | `ComfyUI/models/unet/` | 9.65GB |
| `nsfw_wan_umt5-xxl_fp8_scaled.safetensors` | `ComfyUI/models/text_encoders/` | 約6.7GB（OsriversのNSFW版UMT5） |
| `wan_2.1_vae.safetensors` | `ComfyUI/models/vae/` | 0.25GB（14B用。5B用の `wan2.2_vae` と混同しないこと） |

ファイル名は2026-11-21にHF APIで確認済み。リポジトリのルートにあり、サブフォルダはない。

## 構成

- ベース: `runpod/worker-comfyui:5.8.4-base`（Remixキットと同じ）
- カスタムノード: Remixキットと同じ4件＋ `city96/ComfyUI-GGUF`（ピン `6ea2651e`、2026-01-12）
- モデル: ビルド時に `download_models.sh` でHFから指定ファイルだけダウンロードしてイメージに焼き込む
- GPU: Remixと同じ **L40S（48GB）** で開始。設定の目安も同じ（Max Workers 3、Idle Timeout 5秒、Execution Timeout 1800秒、FlashBoot ON）
- EndpointはRemix（`wan22-remix-14b`）とは別に新規作成する

## 使い方

### 1. `.env` を作る

```bash
cp .env.example .env
# HF_TOKEN は、リポジトリが gated / private の場合のみ必要（公開リポジトリなので不要のはず）
```

### 2. GitHubにプッシュする

Remixキットと同じ手順（`github push-files`）。リポジトリは新規作成すること。

### 3. Endpointを作る

RunPodコンソール → Serverless → New Endpoint → GitHubリポジトリを指定。
GPUは L40S、設定は上記の目安を使う。

### 4. 呼ぶ

`test_payload_same_image.json` を参考に `/run` にPOSTする（Remixと同じテスト画像）。
出力は動画のbase64。本番ではS3/R2に上げる改造を推奨します（`handler.py` のコメント参照）。

```bash
curl -X POST https://api.runpod.ai/v2/<endpoint-id>/run \
  -H "Authorization: Bearer $RUNPOD_API_KEY" \
  -H "Content-Type: application/json" \
  -d @test_payload_same_image.json
```

## 注意

- 成人のみ、フィクションのみ。実在人物のディープフェイクには使わないこと。
- `api-workflow.json` はサーバーレス専用。画面用の `workflow.json`（UI版）はまだ作っていない（`build_workflow_ui.py` はこのキットに含めていない）。画面で開きたい場合は後で生成すること。
- `handler.py` は Remixキットと同じもの（`api-workflow.json` を自動で読む）。
- Q6_K版（`...FASTMOVEQ6KHigh/Low.gguf`、各12GB）もあるが、軽さ優先でQ4_K_M V2を選んだ。画質が気になる場合は差し替え可能。
