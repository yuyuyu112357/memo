---
tags:
  - "#AUTOSAR"
  - "#CRYPTO"
  - "#SecOC"
  - "#autosar-cryptostack"
  - "#crypto-service-manager"
  - "#crypto-stack"
  - "#autosar-csm"
  - "#cryif"
---
# Q&A
## Q1.
>ECU 抽象層。CSM で受けたジョブを、実装能力を持つ Crypto Driver へルーティングするスイッチ役。複数ドライバを同時サポートするため **Channel** という概念でマッピングを行う。([ignitarium.com][1], [charliegearstech.github.io][2])

ルーティングやマッピングとは、どのような意味でしょうか?
## A1.
### CryptoStack における「ルーティング」と「マッピング」の違いと役割

| 用語                   | 何を決める？                                   | 主担当モジュール                                          | いつ行われる？                       | 具体例                                             | イメージ                                 |
| -------------------- | ---------------------------------------- | ------------------------------------------------- | ----------------------------- | ----------------------------------------------- | ------------------------------------ |
| **ルーティング (Routing)** | **どの暗号ドライバ**へジョブを“流す”か                   | **CryIf**（Crypto Interface）                       | **実行時**。アプリ／CSM からジョブが届くたびに判定 | 「AES-GCM は HSM ハードウェアで、SHA-256 はソフトライブラリで動かす」   | 水道の分岐バルブ：流れてきた水（ジョブ）を適切なパイプ（ドライバ）へ送る |
| **マッピング (Mapping)**  | **どの経路（チャネル）**を使ってルーティングするかを **事前に対応付け** | **設定ツール & ARXML**（CryIf/CSM/Crypto Driver のコンフィグ） | **ビルド時**。ECU の能力や要件を見て開発者が設定  | 「Channel 0＝AES/SHA を HW、Channel 1＝RSA/ECC を SW」 | 地図の凡例：道路（チャネル）をあらかじめ“ここを通る”と決めておく    |

---
#### 1. ルーティング ― 実行時に決まる“振り分け”

1. **CSM がジョブ生成**    
    - ジョブには _Primitive_（例：`CsmPrimitive_AESEncrypt`）とキー ID、データ長などが含まれる。        
2. **CryIf がチャネルを参照**    
    - 受け取ったジョブの Primitive をキーに、どの **Channel** を使うかを調べる。        
3. **Crypto Driver へ転送**    
    - チャネルに結び付いているドライバに API 呼び出しを転送。        
4. **結果を逆ルートで返却**    

> **ポイント**：同じ Primitive でも、ECU に複数のドライバがあれば、CryIf が「今空いているほう」や「高速実装」を選ぶといったポリシーも設定で可能。

---
#### 2. マッピング ― 設計段階で決める“経路表”
- **ARXML 例 (抜粋)**
    
    ```xml
    <CryIfChannel ShortName="Channel0">
      <AssignedCryptoDriverRef DEST="CRYPTO-DRIVER">HwDrv</AssignedCryptoDriverRef>
      <AllowedPrimitives>
        <CsmPrimitiveRef DEST="CSM-PRIMITIVE">AESEncrypt</CsmPrimitiveRef>
        <CsmPrimitiveRef DEST="CSM-PRIMITIVE">AESDecrypt</CsmPrimitiveRef>
        <CsmPrimitiveRef DEST="CSM-PRIMITIVE">SHA256</CsmPrimitiveRef>
      </AllowedPrimitives>
    </CryIfChannel>
    ```
    
- **設計者が決める内容**    
    1. ECU に載る **Crypto Driver の種類**（HW/SW、ベンダ依存）        
    2. **Primitive ↔ Channel** の組合せ        
    3. **優先度やキューサイズ**（CSM 側）        

> **ポイント**：マッピングを変更してもアプリ側コードは一切修正不要。“設定だけで切り替えられる”ことが狙い。

---
#### 3. まとめ
- **ルーティング** = **実行時のスイッチング**：CryIf がリアルタイムにジョブを所定ドライバへ振り分けるプロセス。    
- **マッピング** = **設計時のテーブル作り**：どのジョブをどのチャネル（＝ドライバ）へ送るかを ARXML で静的に定義する作業。    

これらを分離することで、
1. **アプリは暗号実装を気にしない**（API だけ呼べば良い）。    
2. **ECU ごとに柔軟に最適化**（HW が無い場合は SW ドライバにマッピング）。    
3. **将来のアルゴリズム追加も安全**（CsmCustom + 新 Driver を追加し、マッピングを書き換えるだけ）。    

つまり CryptoStack の「ルーティング／マッピング」は、
> **“誰が・いつ・どこへ”ジョブを流すのかを、**  
> **実行時（ルーティング）と設計時（マッピング）に二段階で分担する仕組み**

と覚えるとスッキリ理解できます。

## Q2.
>ハード／ソフト実装そのもの。HW アクセラレータ（HSM, SHE, TPM など）や SW ライブラリを透過的に扱う。キー要素・キータイプ定義、ランダム生成、アルゴリズム実装などを提供。([ignitarium.com][1], [charliegearstech.github.io][2])

透過的に扱うとはどのような意味でしょうか?

## A2.
### 「透過的に扱う」の意味
ソフトウェア設計で **“透過的（transparent）”** という言葉が出てきたら、

> **利用者（ここではアプリケーションや上位 BSW）が、背後にある実装の違いを _意識しなくて済む_ 状態**

を指します。別の言い方をすると「ある機能を呼ぶとき、呼び先が何であろうと _見えないガラス越し_ に同じように使える」イメージです。

---
#### CryptoStack での具体例
| |ハードウェア実装 (HSM など)|ソフトウェア実装 (暗号ライブラリ)|
|---|---|---|
|**物理的な場所**|ECU 内の専用回路|MCU の汎用コア上で実行|
|**性能**|高速・鍵が安全領域|CPU 負荷を食うが汎用的|
|**アプリ側の呼び方**|`Csm_AESEncrypt(jobId, dataPtr, keyId)`|**同じ** `Csm_AESEncrypt(jobId, dataPtr, keyId)`|

- **CryIf + Mapping** が「このジョブは HW に」「あのジョブは SW に」と振り分ける    
- アプリや SecOC など **呼び出し側のコードは一切変えない**

つまり **“透過的”** とは、

> 「バックエンドに HW が付くか SW が付くか気にせず、常に同じ API／フローで呼び出せる」  
> という設計上のメリットを示しています。

---
#### 透過性がもたらすメリット

1. **移植容易**    
    - HW アクセラレータの無い開発ボード → 量産 ECU（HW 搭載）へ移る際、アプリの再ビルドだけで済む。
2. **性能最適化が後回しにできる**    
    - まず SW 実装で開発を進め、量産フェーズで「AES は HW、RSA は SW」など最適化してもアプリ修正不要。
3. **セキュリティアップデートが容易**    
    - 新しい暗号ライブラリやパッチを組み込んでも、CryIf のマッピングを切り替えるだけで反映できる。

---
#### まとめワンフレーズ

> **CryptoStack が暗号ドライバを “透過的に扱う” とは、**  
> **アプリ側から見ると HW/SW の境界がガラス張りで存在を感じさせない――**  
> **呼び出しインタフェースは常に同じで、裏側の実装差異を CryIf + マッピングが吸収してくれる、という意味です。**


## Q3.
> **処理フロー例**
> アプリ → `Csm_AESEncrypt()` → CSM がジョブ生成 → CryIf がチャネル判定 → Crypto Driver が AES 処理 → 結果を CryIf 経由で CSM へ返却 → アプリコールバック／戻り値

結果の返却について、アプリコールバック／戻り値の2通りの方法があるのはなぜでしょうか? 私の推測では、CSMは同期、非同期の2つの処理があり、非同期の場合は戻り値が受け取れないため、コールバックを使用すると推測します。

## A3.
ご推察の **「同期＝戻り値、非同期＝コールバック」** は概ねそのとおりですが、AUTOSAR CSM が 2 通りを用意している背景には、もう少し細かい事情があります。ポイントを 4 つに整理してみます。

---
## 1. 同期処理 ― その場で完結できるジョブ

| 特徴                  | 内容                                                                   |
| ------------------- | -------------------------------------------------------------------- |
| **所要時間が短い**         | 典型的には数十 µs〜数 ms で終わるハッシュや乱数取得など。                                     |
| **API 戻り値に結果ステータス** | `Std_ReturnType` で `E_OK/E_NOT_OK` を返す。データそのものは呼び出し時に渡したバッファへ直書きされる。 |
| **タスクをブロック**        | 呼び出しタスクが完了を待つため、リアルタイム性を犠牲にしない長さに限定。                                 |

> CSM 仕様では「同期／非同期モードは静的にコンフィグできる」ことが要求されています。([thietbivanphong.shophoahue.com](https://thietbivanphong.shophoahue.com/AUTOSAR/AUTOSAR_4_2/Software-Architecture_Safety-and-Security/standard/AUTOSAR_SWS_CryptoServiceManager.pdf?utm_source=chatgpt.com "[PDF] Document Title Specification of Crypto Service Manager"))

---
## 2. 非同期処理 ― 時間がかかる or 外部 HW を叩くジョブ

| 特徴                     | 内容                                                           |
| ---------------------- | ------------------------------------------------------------ |
| **長時間・割り込み駆動**         | RSA 2048 bit 署名、AES-GCM を HSM で処理、DMA 転送を伴う場合など。             |
| **API 戻り値は “受付けた” だけ** | 典型的には `E_PENDING`（あるいは `E_OK`＋ジョブ ID）で直帰。実データはまだ無い。          |
| **完了通知は 2 種類**         | ① **コールバック関数**（最も一般的） ② **ポーリング関数**（`Csm_GetJobResult()` など） |
| **CPU をブロックしない**       | 上位タスクはすぐ別処理へ移れる。                                             |

> CSM は「ジョブ完了を上位に通知するコールバック・ポート」を定義しています。([thietbivanphong.shophoahue.com](https://thietbivanphong.shophoahue.com/AUTOSAR/AUTOSAR_4_2/Software-Architecture_Safety-and-Security/standard/AUTOSAR_SWS_CryptoServiceManager.pdf?utm_source=chatgpt.com "[PDF] Document Title Specification of Crypto Service Manager"))

---
## 3. “2 通り” を持つ理由

|理由|説明|
|---|---|
|**リアルタイム制御との両立**|周期 1 ms の制御タスクから AES を呼び出したい場合、同期だと WCET を超える恐れ。非同期＋コールバックならタスクを遅延させない。|
|**ハード／ソフト混在**|ソフト実装しか無い開発ボードでは同期で十分 → 量産 ECU で HSM を使うと非同期が必須…といった **移植性** を確保。|
|**スケジューリング最適化**|複数クライアントが CryptoStack を共有すると、CSM のジョブキューで優先度制御が必要。非同期モードなら CSM が OS タスクのようにキューを捌ける。|
|**API 一貫性**|どちらのモードでも **同じ関数名**（例 `Csm_AESEncrypt()`）で呼び出せる。コンフィグだけで動作モードを切替えられる “透過性” を保つ。|

---
## 4. 補足 — 非同期コールバックの実際の流れ

1. **呼び出し**    
    ```c
    Std_ReturnType ret = Csm_AESEncrypt(jobId, inPtr, outPtr);
    // ret == E_PENDING
    ```
    
2. **CSM_MainFunction()（周期関数）** がジョブキューを回し CryIf → CryptoDrv へ転送    
3. **Crypto Driver 完了 → CSM へ戻る**    
4. **CSM がコールバック呼出し**    
    ```c
    void MyApp_CsmCallback(JobResultType result)
    {
        if (result == CSM_JOB_OK) { /* outPtr に暗号化データが入っている */ }
    }
    ```
    
> 仕様書 SWS_Csm_00032 で、非同期モード選択時はコールバック実装が必須と定義されています。([thietbivanphong.shophoahue.com](https://thietbivanphong.shophoahue.com/AUTOSAR/AUTOSAR_4_2/Software-Architecture_Safety-and-Security/standard/AUTOSAR_SWS_CryptoServiceManager.pdf?utm_source=chatgpt.com "[PDF] Document Title Specification of Crypto Service Manager"))

---
### まとめ
- **戻り値方式** … 処理が短く完結 → 単純・高速だが呼び出し側をブロック    
- **コールバック方式** … 処理が長い／HW 連携 → ノンブロッキングでリアルタイム性を維持    

CryptoStack は ECU ごとの制約や最適化方針に合わせ、**同一 API で両方選べるよう設計** されています。ユーザ（アプリ／他 BSW）はコンフィグを変えるだけで最適な実行形態を採れる――これが 2 通りが併存する理由です。