tags:
	#literature 
	#refactoring
	#file-ops
	#design-pattern 
source:
	[[アプリ上のファイル操作処理方法は？]]
	OpenAI o3

---
# Q&A
## Q1.
ファイル操作方法の一般的な方法は?

### A1.
一般的な整理の仕方
	1. モジュール関数方式
	   ファイル操作ごとに関数を作る
	2. クラス方式
	   FileStorageクラスなどに集約する
	3. インターフェース+実装方式
	   Strategy/Adaptorでストレージ種別を切り替える

## Q2.
クラス方式の場合、staticメソッドが大半にならないか?

### A2.
base_pathやencodingなど「環境依存のパラメータ」を持たせれば、
インスタンスメソッドが自然になる

## Q3.
staticメソッドが大半の場合、純粋関数と変わらないのでは?

### A3.
事実上、その通り
利点は「名前空間」と「インターフェース準拠」のみ

## Q4.
クラス方式のメリットは?

### A4.
* 複数実装(ローカル/クラウドなど)の差しけに強い
* DI(依存関係注入)しやすくテストが書きやすい
* ライフサイクル管理(接続確立/後始末など)を持たせられる

## Q5.
クラス以外の実装方法は?

### A5.
* 単純なユーティリティ関数群（ただし名前衝突に注意）
* fsspec、pathlib、pydantic-settings など既成ライブラリのラッパ
* 関数＋設定用 dataclass を組み合わせる（FP 寄りの設計）

## Q6.
アプリを構成するフォルダや(ソースコードを除く)ファイルのパス情報は
どのような形式で設定するか?

### A6.
* 小規模
	  コードに直書き（定数）
* 中規模以上／複数環境
	  設定ファイル（TOML/YAML/JSON）
* CI・コンテナ
	  環境変数
* CLI ツール
	  コマンドライン引数
* 高度
	  リモート設定サービス、DB など

実務では「CLI > 環境変数 > 設定ファイル > デフォルト」の優先度チェーンで
多段上書きする形が便利

以下、サンプルコード

#### 設定ファイル例
config.toml
```toml
[paths]
input_dir  = "~/data/input"
backup_dir = "~/data/backup"
trash_dir  = "~/data/trash"
```

#### 設定読み込み（環境変数・CLI で上書き可）
config.py
```python
from pathlib import Path
from typing import Annotated
from pydantic import BaseModel, Field
import os
import tomllib

class PathConfig(BaseModel):
    input_dir:  Annotated[Path, Field(alias="INPUT_DIR")]
    backup_dir: Annotated[Path, Field(alias="BACKUP_DIR")]
    trash_dir:  Annotated[Path, Field(alias="TRASH_DIR")]

    @classmethod
    def load(cls, file_path: Path | str, cli_overrides: dict[str, str] | None = None) -> "PathConfig":
        # 1) TOML
        data: dict = {}
        if Path(file_path).exists():
            with open(file_path, "rb") as fp:
                data = tomllib.load(fp).get("paths", {})
        # 2) 環境変数
        env = {k: v for k, v in os.environ.items() if k in {"INPUT_DIR", "BACKUP_DIR", "TRASH_DIR"}}
        # 3) CLI 上書き
        merged = {**data, **env, **(cli_overrides or {})}
        return cls.model_validate(merged)

    def expand(self) -> "PathConfig":
        return PathConfig(
            input_dir=self.input_dir.expanduser().resolve(),
            backup_dir=self.backup_dir.expanduser().resolve(),
            trash_dir=self.trash_dir.expanduser().resolve(),
        )
```

#### ファイル操作ラッパ
file_manager.py
```python
from pathlib import Path
import shutil
from config import PathConfig

class FileManager:
    def __init__(self, cfg: PathConfig) -> None:
        self.cfg = cfg.expand()

    def copy_to_backup(self, rel_path: str | Path) -> Path:
        src = self.cfg.input_dir / rel_path
        dst = self.cfg.backup_dir / rel_path
        dst.parent.mkdir(parents=True, exist_ok=True)
        return Path(shutil.copy2(src, dst))

    def move_to_trash(self, rel_path: str | Path) -> Path:
        src = self.cfg.input_dir / rel_path
        dst = self.cfg.trash_dir / rel_path
        dst.parent.mkdir(parents=True, exist_ok=True)
        return Path(shutil.move(src, dst))
```

#### エントリポイント
main.py
```python
import argparse
from pathlib import Path
from config import PathConfig
from file_manager import FileManager

def _parse_args() -> argparse.Namespace:
    ap = argparse.ArgumentParser()
    ap.add_argument("--config", default="config.toml")
    ap.add_argument("--input-dir")
    ap.add_argument("--backup-dir")
    ap.add_argument("--trash-dir")
    ap.add_argument("filename")
    return ap.parse_args()

def main() -> None:
    args = _parse_args()
    overrides = {k.upper(): v for k, v in vars(args).items()
                 if k in {"input_dir", "backup_dir", "trash_dir"} and v}
    cfg = PathConfig.load(Path(args.config), overrides)
    fm = FileManager(cfg)
    fm.copy_to_backup(args.filename)
    print("✔ backed up:", args.filename)

if __name__ == "__main__":
    main()

```
