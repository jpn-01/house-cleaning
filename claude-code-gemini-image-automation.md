# Claude CodeとGeminiを連携させた画像生成自動化の方法

## はじめに

Claude CodeとGoogle Geminiを組み合わせることで、テキスト指示から画像生成までのパイプラインを完全に自動化できます。本記事では、Claude Codeをオーケストレーター（司令塔）として使い、Gemini APIの画像生成機能（Imagen 3）を呼び出す実装手順を解説します。

---

## 全体アーキテクチャ

```
ユーザーの指示
    │
    ▼
Claude Code（プロンプト整形・ワークフロー制御）
    │
    ▼
Gemini API（Imagen 3による画像生成）
    │
    ▼
生成画像をローカル保存 / クラウドアップロード
```

Claude Codeが「考える役」、GeminiのImagen 3が「描く役」です。Claude Codeはユーザーの曖昧な指示を詳細なプロンプトに変換し、Gemini APIへリクエストを送ります。

---

## 前提条件

| 項目 | 内容 |
|------|------|
| Claude Code | Claude Code CLI インストール済み |
| Google Cloud / AI Studio | Gemini APIキー取得済み |
| Python | 3.10以上 |
| ライブラリ | `google-generativeai` >= 0.8.0 |

---

## ステップ1：環境セットアップ

### 1-1. Gemini APIライブラリのインストール

```bash
pip install google-generativeai pillow requests
```

### 1-2. APIキーの設定

```bash
# .envファイルに記載（リポジトリにはコミットしない）
export GEMINI_API_KEY="your_api_key_here"
```

---

## ステップ2：Gemini画像生成スクリプトの作成

`generate_image.py` を作成します。

```python
import os
import base64
from pathlib import Path
import google.generativeai as genai
from PIL import Image
import io

# APIキー設定
genai.configure(api_key=os.environ["GEMINI_API_KEY"])

def generate_image(prompt: str, output_path: str = "output.png") -> str:
    """
    Gemini Imagen 3を使って画像を生成し、ファイルに保存する。
    
    Args:
        prompt: 画像生成プロンプト（英語推奨）
        output_path: 保存先ファイルパス
    
    Returns:
        保存したファイルのパス
    """
    model = genai.ImageGenerationModel("imagen-3.0-generate-001")
    
    result = model.generate_images(
        prompt=prompt,
        number_of_images=1,
        safety_filter_level="block_only_high",
        person_generation="allow_adult",
        aspect_ratio="1:1",
        language="auto",
    )
    
    image = result.images[0]
    
    # PIL Imageとして保存
    pil_image = Image.open(io.BytesIO(image._pil_image.tobytes()))
    pil_image.save(output_path)
    
    return output_path


if __name__ == "__main__":
    import sys
    prompt = sys.argv[1] if len(sys.argv) > 1 else "A futuristic cityscape at night"
    path = generate_image(prompt)
    print(f"Image saved: {path}")
```

---

## ステップ3：Claude Codeによるプロンプト自動整形

Claude Codeをオーケストレーターとして使い、日本語の指示を英語の詳細プロンプトに変換します。

`orchestrator.py` を作成します。

```python
import anthropic
import subprocess
import os
import sys

client = anthropic.Anthropic()


def refine_prompt_with_claude(user_instruction: str) -> str:
    """
    Claude Codeを使ってユーザーの指示を詳細な英語プロンプトに変換する。
    """
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[
            {
                "role": "user",
                "content": f"""以下のユーザー指示を、Imagen 3（テキストから画像生成AI）向けの
詳細な英語プロンプトに変換してください。

ルール：
- 英語のみで出力する
- スタイル、構図、色調、ライティングを具体的に記述する
- 150ワード以内にまとめる
- プロンプト本文のみ出力する（前置き・説明不要）

ユーザー指示：
{user_instruction}""",
            }
        ],
    )
    return response.content[0].text.strip()


def run_pipeline(user_instruction: str, output_path: str = "output.png") -> str:
    """
    エンドツーエンドのパイプラインを実行する。
    1. Claudeでプロンプト整形
    2. Geminiで画像生成
    3. ファイル保存
    """
    print(f"[1/3] Claudeでプロンプト整形中...")
    refined_prompt = refine_prompt_with_claude(user_instruction)
    print(f"生成プロンプト:\n{refined_prompt}\n")

    print(f"[2/3] Geminiで画像生成中...")
    result = subprocess.run(
        ["python", "generate_image.py", refined_prompt, output_path],
        capture_output=True,
        text=True,
    )

    if result.returncode != 0:
        raise RuntimeError(f"画像生成失敗: {result.stderr}")

    print(f"[3/3] 完了: {result.stdout.strip()}")
    return output_path


if __name__ == "__main__":
    instruction = sys.argv[1] if len(sys.argv) > 1 else "夕焼けの海岸で波が打ち寄せる幻想的な風景"
    output = sys.argv[2] if len(sys.argv) > 2 else "output.png"
    run_pipeline(instruction, output)
```

---

## ステップ4：Claude Codeのツール機能を活用した高度な自動化

Claude Codeのツール呼び出し（Tool Use）を使うと、会話の中で自動的に画像生成を実行できます。

```python
import anthropic
import json
import subprocess

client = anthropic.Anthropic()

# Claude Codeに渡すツール定義
tools = [
    {
        "name": "generate_image",
        "description": "テキストプロンプトから画像を生成してファイルに保存する",
        "input_schema": {
            "type": "object",
            "properties": {
                "prompt": {
                    "type": "string",
                    "description": "英語の画像生成プロンプト",
                },
                "output_filename": {
                    "type": "string",
                    "description": "保存するファイル名（例: sunset.png）",
                },
            },
            "required": ["prompt", "output_filename"],
        },
    }
]


def execute_tool(tool_name: str, tool_input: dict) -> str:
    if tool_name == "generate_image":
        result = subprocess.run(
            ["python", "generate_image.py", tool_input["prompt"], tool_input["output_filename"]],
            capture_output=True,
            text=True,
        )
        return result.stdout if result.returncode == 0 else f"Error: {result.stderr}"
    return "Unknown tool"


def chat_with_image_generation(user_message: str):
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )

        if response.stop_reason == "end_turn":
            print(response.content[0].text)
            break

        if response.stop_reason == "tool_use":
            # ツール呼び出しを実行
            tool_use = next(b for b in response.content if b.type == "tool_use")
            print(f"ツール実行中: {tool_use.name}({tool_use.input})")
            tool_result = execute_tool(tool_use.name, tool_use.input)

            messages.append({"role": "assistant", "content": response.content})
            messages.append({
                "role": "user",
                "content": [{"type": "tool_result", "tool_use_id": tool_use.id, "content": tool_result}],
            })


if __name__ == "__main__":
    chat_with_image_generation(
        "神戸の夜景をイメージした美しい画像を3枚生成してください。"
        "ファイル名は kobe_night_1.png、kobe_night_2.png、kobe_night_3.png にしてください。"
    )
```

---

## ステップ5：バッチ自動化スクリプト

複数の画像を一括生成する場合は、YAMLで指示リストを管理します。

`batch_config.yaml` の例：

```yaml
images:
  - instruction: "神戸港の夜景、ルミナリエのイルミネーション付き"
    output: "kobe_port_night.png"
  - instruction: "有馬温泉の旅館、紅葉シーズン、秋の朝霧"
    output: "arima_onsen_autumn.png"
  - instruction: "六甲山からの大阪湾の眺め、晴天の昼間"
    output: "rokko_view.png"
```

`batch_generate.py` の例：

```python
import yaml
from orchestrator import run_pipeline

with open("batch_config.yaml") as f:
    config = yaml.safe_load(f)

for item in config["images"]:
    print(f"\n--- 生成開始: {item['output']} ---")
    path = run_pipeline(item["instruction"], item["output"])
    print(f"保存完了: {path}")
```

実行コマンド：

```bash
python batch_generate.py
```

---

## ポイントまとめ

| 役割 | 使用技術 | 担当内容 |
|------|----------|----------|
| オーケストレーター | Claude Code（claude-sonnet-4-6） | 日本語指示→英語プロンプト変換、ワークフロー制御 |
| 画像生成エンジン | Gemini Imagen 3 | テキストから高品質画像を生成 |
| 実行基盤 | Python / subprocess | スクリプト間の連携・ファイル管理 |

---

## セキュリティ上の注意

- APIキーは必ず環境変数または `.env` ファイルで管理し、Gitリポジトリにコミットしない
- `.gitignore` に `.env` と `*.png`（生成画像）を追加する
- Imagen 3の利用規約を確認し、商用利用の可否を必ず確認すること

---

## まとめ

Claude CodeとGeminiを組み合わせることで、

1. **日本語の曖昧な指示** → Claudeが英語の詳細プロンプトに整形
2. **整形済みプロンプト** → Imagen 3が高品質な画像を生成
3. **バッチ処理対応** → YAMLで複数画像を一括自動生成

というパイプラインを簡単に構築できます。Claude Codeのツール呼び出し機能を活用すれば、会話形式でインタラクティブに画像生成を制御することも可能です。ぜひ試してみてください。
