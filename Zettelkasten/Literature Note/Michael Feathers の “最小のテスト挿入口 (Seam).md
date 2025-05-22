tags:
	#literature 
	#refactoring 
	#unit-test 
	#testability 
source:
	[[リファクタリングに取り組む順序]]
	OpenAI o3

---
# Q&A
## Q1.
>* Michael Feathers の “最小のテスト挿入口 (Seam) を作る → テストを書く → 安心して小リファクタリング”  

とは、何か?

### A.1
Michael Feathers の著書  
“Working Effectively with Legacy Code” が提唱する核心アイデアのひとつが

 最小のテスト挿入口（Seam）を作る → テストを書く → 安心してリファクタリングする

という３ステップです。  
以下では

1. Seam とは何か
2. Seam を作るための代表テクニック
3. Python での具体例（Before → Seam 作成 → テスト → リファクタリング）
4. 実務での運用フロー

を順に説明します。

## Q2.
Seam とは？

### A2.
Feathers の定義：
    「ソースコードに手を入れずに、プログラムの振る舞いを差し替えられる“隙間”」

要するに

• 振る舞いを注入（インジェクション）できるポイント  
• そこを利用して外部依存や副作用をスタブ／モックに置き換え、テストを書く

という“注射口”です。

## Q3.
代表的な Seam の作り方は?

### A3.
(書籍の分類 + Python 的補足を併記)

a. オブジェクト Seam  
– 依存オブジェクトをコンストラクタや関数引数で渡す（DI）  
– Python だと引数にデフォルト値を与え、テストで差し替えるのが簡単

b. メソッド Seam（Extract & Override）  
– 親クラスのメソッドを抽出し、テスト用サブクラスでオーバーライド

c. ファクトリ Seam  
– “内部で直接 new する”のをやめ、生成責務を別関数／ファクトリに隔離

d. ラッパ Seam（Sprout Class / Sprout Method）  
– 既存関数の周囲に薄いラッパを生やし、副作用部分を切り離す

e. モンキーパッチ Seam（動的差し替え）  
– Python の `unittest.mock.patch` や Monkeypatch でランタイム上書き

## Q4.
具体的な使い方は?

### A4.
▼1 まずは「テスト不能」なレガシー関数
before.py
```python
import datetime

def archive_today(src_path: str, dst_dir: str) -> str:
    """
    src_path のファイルを 'YYYYMMDD_HHMMSS.txt' へリネームし
    dst_dir にコピーして戻り値として新パスを返す
    """
    ts = datetime.datetime.now()           # ← 時刻依存、副作用！
    new_name = ts.strftime("%Y%m%d_%H%M%S.txt")

    with open(src_path, "rb") as f_src:    # ← ファイル I/O
        data = f_src.read()

    dst_path = f"{dst_dir}/{new_name}"
    with open(dst_path, "wb") as f_dst:
        f_dst.write(data)

    return dst_path
```

課題  
• `datetime.now()` とファイル I/O のせいで単体テストが書きづらい  
• 振る舞いを確認するには実ファイルを書き、時刻も固定できない

▼2 最小限の変更で Seam を挿入（注：振る舞いは変えない）
with_seam.py
```python
import datetime
from typing import Callable, BinaryIO

TimeProvider = Callable[[], datetime.datetime]

def archive_today(
    src_fp: BinaryIO,
    dst_writer: Callable[[bytes, str], None],
    clock: TimeProvider = datetime.datetime.now,
) -> str:
    ts = clock()                                   # ← 時間依存を注入
    new_name = ts.strftime("%Y%m%d_%H%M%S.txt")

    data = src_fp.read()                           # ← src_fp も注入

    dst_writer(data, new_name)                     # ← 書き込みロジックも注入
    return new_name
```

変更点  
• ファイルパスではなく「読み込み済みファイルオブジェクト」を引数に  
• 書き込みは `dst_writer(data, new_name)` というコールバックに委譲  
• 時刻取得は `clock()` で外から差し替え可能  
→ たった３カ所の局所変更で、I/O と時間が丸ごと差し替え可能な Seam になった

▼3 テストを書く
test_archive_today.py
```python
import io
from legacy.with_seam import archive_today
import datetime

def fake_clock():
    return datetime.datetime(2025, 1, 2, 3, 4, 5)

def fake_writer(data: bytes, name: str):
    fake_writer.captured = (data, name)   # テスト用に保存
fake_writer.captured = None

def test_archive_today():
    src = io.BytesIO(b"hello")
    new_name = archive_today(src, fake_writer, fake_clock)

    assert new_name == "20250102_030405.txt"
    saved_data, saved_name = fake_writer.captured
    assert saved_data == b"hello"
    assert saved_name == "20250102_030405.txt"
```

• ファイルを `io.BytesIO` で置き換え → ディスク不要  
• 時刻を固定 `fake_clock()` → 決 determin 性確保  
• `dst_writer` もメモリ内スタブ

この時点で「副作用ゼロ」の高速テストが完成。

▼4 安心して本格リファクタリング

Seam があるので以下のような変更も安全にできる：

- ファイルコピー部を専用クラスに抽出
- フォーマットを `pathlib` + 型安全に変更
- エラーハンドリングを追加、など

テストが意図を守ってくれるためリファクタリングの自由度が上がります。

## Q5.
実務フローは?

### A5.
1. 変更したい関数／クラスを選ぶ（バグ修正や機能追加のホットスポット）
2. テストが書きにくい原因（副作用・グローバル・時間依存等）を洗い出す
3. 「最小の行数」で Seam を作成  
    引数追加 or コールバック注入 or メンバー関数抽出 のいずれかで済ます
4. Seam を利用したテストを追加  
    Red → Green を確認（まだリファクタリングはしない）
5. テストが緑のまま振る舞いを変えずにリファクタリング
6. テストが守っている範囲を少しずつ広げながら (2) へ戻る

ポイント  
・“最小” ＝ ビルドやレビューを通る最小単位。大改造は厳禁  
・初回は 5～20 行の変更 + 1 ～ 2 件のテスト から始めると失敗しにくい  
・Seam の挿入自体も「テスト対象の振る舞いを壊さない」ことが最重要

## まとめ
* Seam は「副作用や外部依存を差し替えるための注射口」  
* まず Seam を生やしてテストを書き、振る舞いをロックする  
* その後のリファクタリングはテストが安全網になるため大胆に行える

この“Seam → テスト → リファクタリング”のサイクルを繰り返すことで、  
テストがなかったレガシーコードでも段階的に健全な設計へと移行できます。
