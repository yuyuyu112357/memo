---
tags:
  - "#AUTOSAR"
  - "#CRYPTO"
  - "#SecOC"
  - "#autosar-cryptostack"
  - "#crypto-service-manager"
---
### AUTOSAR CryptoStack 概要

**1. そもそも CryptoStack とは？**
CryptoStack は AUTOSAR が規定する暗号サービス群の総称で、ECU（電子制御ユニット）上のアプリケーションや他の BSW モジュール（SecOC、診断、ブートローダなど）から暗号アルゴリズムを *標準 API* で呼び出せるようにするための基盤です。これにより **機密性・完全性・真正性** といったセキュリティ目標を ECU 間通信やファームウェア更新、鍵管理などの場面で実現できます。([ignitarium.com][1])

---

**2. Classic Platform における 3 層アーキテクチャ**

| レイヤ                              | 役割（かぎ括弧は代表 API／設定項目）                                                                                                                                                         |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Crypto Service Manager (CSM)** | サービス層。アプリ／他 BSW からの暗号リクエストを「ジョブ」として受け取り、優先度付きキューで管理。ジョブは「プリミティブ（hash, AES, ECDSA など）」と「キー」「データ長」等の属性を持つ。同期／非同期処理を選択可能。([ignitarium.com][1], [charliegearstech.github.io][2]) |
| **Crypto Interface (CryIf)**     | ECU 抽象層。CSM で受けたジョブを、実装能力を持つ Crypto Driver へルーティングするスイッチ役。複数ドライバを同時サポートするため **Channel** という概念でマッピングを行う。([ignitarium.com][1], [charliegearstech.github.io][2])                |
| **Crypto Driver (Crypto)**       | ハード／ソフト実装そのもの。HW アクセラレータ（HSM, SHE, TPM など）や SW ライブラリを透過的に扱う。キー要素・キータイプ定義、ランダム生成、アルゴリズム実装などを提供。([ignitarium.com][1], [charliegearstech.github.io][2])                         |
[[AUTOSAR CryptoStack 概要_Q&A#Q1.]]
[[AUTOSAR CryptoStack 概要_Q&A#Q2.]]


> **処理フロー例**
> アプリ → `Csm_AESEncrypt()` → CSM がジョブ生成 → CryIf がチャネル判定 → Crypto Driver が AES 処理 → 結果を CryIf 経由で CSM へ返却 → アプリコールバック／戻り値

[[AUTOSAR CryptoStack 概要_Q&A#Q3.]]

---

**3. Adaptive Platform との違い**

Adaptive では **Provider** という役割分担に変わります。代表的には

* **Crypto Provider (CP)**：アルゴリズム実行
* **Key Storage Provider (KSP)**：安全な鍵格納
* **X.509 Provider (CMP)**：証明書／CRL 管理
* **Objects Coding Provider (OCP)**：DER／PEM 等の符号化変換

モジュール間を ara::crypto API で束ねる点は同じですが、マイクロサービス的にスケール出来る構造が特徴です。([linkedin.com][3])

---

**4. サポートされる暗号プリミティブ（代表例）**

* **対称鍵**：AES (ECB/CBC/GCM) ほか
* **非対称鍵**：RSA, ECDSA, Ed25519 など
* **ハッシュ／MAC**：SHA-2/3, HMAC, CMAC
* **乱数生成**：TRNG, DRBG
* **鍵操作**：生成・導出・交換 (DH/ECDH)
* **署名／検証**、**AEAD** など
  これらは CsmPrimitive としてリクエスト可能で、ジョブ単位でパラメータ（キー長、データ長、同期／非同期 など）を指定します。([charliegearstech.github.io][2], [linkedin.com][3])

---

**5. 主なユースケース**

| ユースケース                                    | CryptoStack が担う機能                                                |
| ----------------------------------------- | ---------------------------------------------------------------- |
| **Secure On-Board Communication (SecOC)** | PDU へ MAC／フレッシュネスカウンタを付与し改ざん検出。([charliegearstech.github.io][2]) |
| **診断サービスの SecurityAccess**                | チャレンジレスポンスの暗号計算、乱数生成。                                            |
| **Secure Boot / Flash & OTA 更新**          | ファームウェア署名検証、暗号化更新パッケージの復号。                                       |
| **TLS／IPsec などネットワークセキュリティ**              | RNG、鍵導出、ハンドシェイク時の署名／検証。                                          |

---

**6. コンフィギュレーションの観点**

* **ARXML** で *CsmPrimitives, CsmJobs, CsmQueues* を定義
* **CryIfChannels** にキュー⇔ドライバの経路をマッピング
* **CryptoDriverObjects / CryptoPrimitives / CryptoKeys** で HW・SW 能力を表現
* 永続鍵は NvM ブロックとリンクし、起動時に読み込み

ベンダ提供の *pre-config* ファイルを読み込んで下位→上位へ設定を積み上げるのが一般的です。([ignitarium.com][1])

---

**7. 最近の仕様動向（R24-11 時点）**

* CSM に **CsmCustom Service** が追加され、標準外アルゴリズムを拡張しやすくなった
* Adaptive 側では **ポスト量子暗号** 評価のためのプレースホルダー API が検討フェーズに
* Classic/Adaptive 共に ISO/SAE 21434、UNECE R155 対応を強化

※ 公開 PDF では詳細が制限されているため、実務では AUTOSAR メンバーシップ経由で最新版仕様を確認してください。([autosar.org][4])

---

#### まとめ

CryptoStack は **「暗号を一元化し、実装依存を排除するレイヤ」** と捉えると理解しやすいです。Classic の 3 層と Adaptive の Provider 群という 2 系統がありますが、いずれも「*アプリは鍵とアルゴリズムを意識せず、サービスとして呼ぶだけ*」という思想は共通です。今後は量子耐性アルゴリズムや高性能 HSM との連携など、さらに発展が見込まれます。

[1]: https://ignitarium.com/software-crypto-stack-for-automotive-ecus/ "Software Crypto Stack for Automotive ECUs | ignitarium.com"
[2]: https://charliegearstech.github.io/blog/Autosar_CyberSecurity/ "Autosar Cyber Security – GearsTech by Carlos Hdez"
[3]: https://www.linkedin.com/pulse/autosar-cryptography-enhancing-security-automotive-pallerla-jmd0e "AUTOSAR Crypto : Enhancing Security in Automotive Applications"
[4]: https://www.autosar.org/fileadmin/standards/R24-11/CP/AUTOSAR_CP_SWS_CryptoServiceManager.pdf?utm_source=chatgpt.com "[PDF] Specification of Crypto Service Manager AUTOSAR CP R24-11"
