tags:
	#literature 
	#refactoring 
	#unit-test 
	#testability 
source:
	[[疑似グローバル変数に関するリファクタリング]]
	OpenAI o3

---
# Q&A
## Q1.
>Step 3 旧 import 部分は ****暫定的に**** `default_context.api_url` などを参照するアダプタに置換

とは?

### A1.
「Step 3：旧 import 部分を暫定的に default_context.xxx へ張り替える」は、

• “昔のコード（グローバル定数を直 import）”を一度に書き換えなくても動かす  
• でも“新しい Context-ベース”の世界へ段階的に移行できる

という橋渡しレイヤを作るフェーズです。  
ポイントは「旧 API を温存しつつ背後で値を Context に寄せる」こと。

Context と default_context の作成

1. Context と default_context の作成

context.py
```python
from __future__ import annotations
from dataclasses import dataclass, replace

@dataclass(frozen=True)
class Context:
    api_url: str
    timeout_sec: int

# アプリ起動時に 1 個だけ生成
default_context = Context(
    api_url="https://api.example.com",
    timeout_sec=3,
)

def override_default(**kwargs) -> None:
    """
    テストや dev サーバ起動時に
    default_context の一部を差し替えるユーティリティ。
    """
    global default_context
    default_context = replace(default_context, **kwargs)
```

2. 旧モジュールを“薄いアダプタ”に差し替える
旧来は `config.py` に定数がベタ書きされていたとします。  
ファイル名を変えず中身だけ次のように張り替えるか、  
新しい `legacy_config.py` を用意して旧コードを順次そちらへ  
import し直す方法があります（ここでは後者を例示）。

legacy_config.py
```python
"""
旧コードが `from legacy_config import API_URL` と書き続けても
裏では Context 由来の値を返す“アダプタ”層。
将来的に削除予定なので、**新しいコードは USE 禁止**。
"""
from __future__ import annotations
from .context import default_context, override_default

def _bind():
    """
    default_context 内の値をモジュール変数へコピー。
    """
    globals()["API_URL"] = default_context.api_url
    globals()["TIMEOUT_SEC"] = default_context.timeout_sec

# 初回 import 時にバインド
_bind()

# default_context を差し替えたあと
# 旧 API 利用コードも値を最新にしたい場合は
# これを呼ぶ。例: テストの fixture で。
def refresh() -> None:
    _bind()
```

ポイント
* ***値は import 時に 1 度だけコピー**
	– 旧コードの大半は「起動時に設定が決まれば OK」なのでこの方が単純  
* default_context を動的に差し替える場合（テストなど）は  
	– `override_default(...); legacy_config.refresh()`
	の 2 行で旧 API も最新値に同期できる  
* アプリ本体のライフサイクル中に頻繁に変更する場合は  
	– `API_URL` などを **定数ではなく関数** にする、
	あるいは `sys.modules` 差し替えで「都度属性参照」を行う  
	– ただしそこまで動的に変わるなら、旧 API を早めに廃止した方が安全

3. 新旧コードが同居する具体例
service.py
```python
import requests
from .context import Context

def fetch_user(uid: str, ctx: Context):
    resp = requests.get(f"{ctx.api_url}/users/{uid}", timeout=ctx.timeout_sec)
    resp.raise_for_status()
    return resp.json()
```

legacy_consumer.py
```python
"""
まだリファクタリング前のモジュール。
昔ながらに定数を import して使っている。
"""
from .legacy_config import API_URL, TIMEOUT_SEC
import requests

def fetch_user_legacy(uid: str):
    resp = requests.get(f"{API_URL}/users/{uid}", timeout=TIMEOUT_SEC)
    resp.raise_for_status()
    return resp.json()
```


4. テストからの利用例
test_legacy_and_new.py
```python
from sample.context import override_default
from sample.legacy_config import refresh
from sample.service import fetch_user
from sample.legacy_consumer import fetch_user_legacy

def test_both_paths(httpserver):
    httpserver.expect_request("/users/42").respond_with_json({"name": "zed"})
    # 1. 新しい default_context へ差し替え
    override_default(api_url=httpserver.url_for(""), timeout_sec=1)
    # 2. 旧 API モジュールに反映
    refresh()

    # 新旧どちらの呼び出しも同じエンドポイントへ飛ぶ
    assert fetch_user("42", ctx=httpserver.app["ctx"])["name"] == "zed"
    assert fetch_user_legacy("42")["name"] == "zed"
```

5. フェーズ完了の判断 & 次ステップ
	1. “旧 API import” の参照箇所を静的解析で数え、ゼロになったら
	2. `legacy_config.py` を削除し、アダプタの役目終了

その後、`Context` をさらに細分化（AppContext / RequestContext 等）  
または DI コンテナに委譲するといった“次のリファクタリング”へ進めます。

## まとめ
* アダプタ層は「既存コードを書き換えない」安心感と 
  「新しい Context 流儀」への段階移行を両立させる緩衝材  
* やることは「旧モジュールの中身を default_context の値で再定義」  
* 動的に差し替える可能性があるなら `refresh()` を用意  
* 参照件数が 0 になり次第、アダプタは速やかに削除して負債を完済

これで Step 3 の“暫定ブリッジ”が安全に運用できます。
