# Linux（CUDA）でのビルドと動作確認

更新日: 2026-11-19

TinyStrata は Windows 専用として配布されています。このメモは、**ソースを変更せずに** Linux の CUDA 環境でビルドして動いた範囲を、確認できた事実だけで記録します。公式の対応表明ではありません。README の対応範囲（NVIDIA Ampere 世代以降、Windows）の外にある構成です。

## 確認した環境

- OS: CachyOS（Arch 系）、カーネル 7.3.4、GCC 16.2.1、CMake
- CUDA Toolkit: 13.4.3（`nvcc` 13.4.92、`/opt/cuda`）。`CMakeLists.txt` は 13.2.0〜13.2.1 を拒否するため、13.1 または 13.2.2 以降を使う
- GPU: GeForce RTX 2060 Max-Q（Turing、CC 7.5、VRAM 6 GiB）。システム RAM 22 GiB
- コミット: `73167a6`（main）。ソースの変更なし

## ビルド手順

既定の `CMAKE_CUDA_ARCHITECTURES` は 86 / 89 / 120 で、Turing（75）を含まない。Turing では明示する。

```bash
export PATH=/opt/cuda/bin:$PATH
cmake -S . -B ../TinyStrata-work/build \
  -DTINYSTRATA_BACKEND=CUDA -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CUDA_ARCHITECTURES=75 -DCMAKE_CUDA_COMPILER=/opt/cuda/bin/nvcc
cmake --build ../TinyStrata-work/build --target tinystrata -j6
```

- `tinystrata` ターゲットは、エラー 0 件でビルドできた（警告は多数）
- GPU を使わない確認は `-DTINYSTRATA_BACKEND=NONE` で configure し、`native_conversations_test`・`native_command_inbox_test`・`unicode_paths_test`・`native_utf8_args_test` をビルド・実行した。4 件とも成功した

## 動作確認

- `tinystrata --help` が起動する
- Qwen3.6-35B-A3B UD-IQ3_XXS（SHA-256 は `serve/model_download.py` の固定値と一致）を `tinystrata --model <GGUF> --bench-pp N --bench-tg N` で読み込み、入力処理・生成とも完走した
- 実行時のエンジンログでは、KV/attention の `auto` は **INT8** に決まり（CC 7.5 にネイティブ FP8 MMA がないため）、INT8 の入力アテンションは FP16 ステージングのフォールバックだった。エキスパートはハイブリッド（一部が VRAM、残りがピン留めした RAM を CPU で計算）で、この配置では decode の CUDA Graph は無効になった
- 実行時は `LD_LIBRARY_PATH=/opt/cuda/lib64` を設定した。設定なしで動くかは未確認

## 未確認

- Qwen3.6 以外（Gemma 4・LFM2.5・Clover・Tiel Coder）。`tinystrata-gemma4`・`tinystrata-lfm2` ターゲットは Linux でビルドしていない
- `--serve`、`run.py`、API・UI、Vision（画像入力）
- `benchmark/run.py` の標準プロトコル。上の実行は `--bench-pp/--bench-tg` による簡易確認で、v1 の結果ではない
- 出力の品質、Ampere 以降の GPU、AMD（HIP）の Linux ビルド
- `install.ps1`・`install-ui.ps1`・`.bat` は Windows 用で、Linux では使えない
- Turing は README の対応範囲外で、上流が Turing での動作を保証しているわけではない。ソース内の Turing 向け分岐（例: `src/qwen35/int8_attention.cu`）は、今回 Qwen の実行で通ったが、他の経路は未検証

## 注意

6 GiB の VRAM は README の最小条件（8 GiB）を下回る。実際の速度は GPU の電力制限・給電などの影響を受けるため、このメモには性能の数値を載せない。
