## 1. 現状分析

### 1.1 基本情報
- **職種**: 車載組み込みエンジニア（セキュリティ分野）
- **経験**: 社会人1年目、要求分析担当
- **勤務地**: 名古屋（希望継続）
- **年収**: 600万円
- **学習時間**: 月30-40時間確保可能
- **志向**: 安定志向、スペシャリスト志向

### 1.2 現在のスキル
- **強み**: Python、Matlab等スクリプト言語
- **業務経験**: Vector DaVinci使用、AUTOSAR CRYPTOモジュール設定
- **弱み**: C/C++、Rust実装経験少、英語力不足（TOEIC600程度）
- **AI経験**: LSTM使用経験あり、オンデバイスAI未経験

### 1.3 AI技術に対する姿勢
- **脅威vs機会**: 4:6
- **不安vs期待**: 5:5
- **時間的プレッシャー**: 高（1-3年でAGI登場予測）
- **学習スタイル**: 実践重視、深掘り志向だが最近目移りしがち

## 2. 戦略フレームワーク

### 2.1 4ステップ判断フレーム

#### Step 0: 前提整理
- **目的**: AI時代でも代替されにくい車載セキュリティ専門性確立
- **目標**: 3年後に自立したエンジニアになる
- **制約**: 名古屋ベース、安定志向、月30-40h学習時間

#### Step 1: 必要スキルセット表
**技術層**
- ★ 暗号理論・実装（AES, ECC, TLS, IPsec, HSM, Secure Boot）
- ★ AUTOSAR SecOC/CRYPTO スタック構成 & Vector DaVinci操作
- ☆ C/C++ 組み込み実装＋RTOS
- ☆ Rust（セーフティ分野で台頭）

**プロセス層**
- ★ ISO/SAE 21434, UNECE R155/156, TARA施策
- ☆ ASPICE、機能安全 ISO 26262

**周辺層**
- ★ 英語読解（標準・論文・海外カンファ）
- ☆ AI活用（LLMを用いた要求解析・テスト生成）

#### Step 2: 現在地評価（★5段階）
- 暗号理論: ★1
- C/C++: ★1
- ISO21434: ★0
- AUTOSAR: ★2（業務経験あり）

#### Step 3: 学習バックログ作成
- TrelloやNotionでカンバン化
- 各タスク3-6h単位で小粒化
- 月末バーンダウンレビュー

## 3. 具体的アクションプラン

### 3.1 Phase 1: 基盤固め（0-3ヶ月）
**実行方式**: 1ヶ月スプリント × 週次タスクで運用

#### 3.1.1 ISO/SAE 21434 & TARA 基礎
- [ ] 書籍『自動車サイバーセキュリティ入門』（JSAE）通読・要約（A4 2枚）
- [ ] Vector無料ウェビナー or TÜV SÜD 1 Day オンライン講習受講

#### 3.1.2 C言語リフレッシュ & 暗号実装
- [ ] STM32 Nucleo + mbedTLS でAES256-CTRベンチマーク測定
- [ ] 実測結果レポート作成（A4 1枚、業務説得材料として）

#### 3.1.3 英語力強化（読解重視）
- [ ] IETF RFC 8998 (HSS/LMS) をAI翻訳併用で通読
- [ ] 重要フレーズをAnkiで暗記

#### 3.1.4 学習インフラ整備
- [ ] Obsidianで Zettelkasten ノート体系作成
- [ ] GitHub Copilot/Claude/ChatGPT使い分け手順書作成

### 3.2 Phase 2: 差別化（3-12ヶ月）

#### 3.2.1 21434実務スキル化
- [ ] 擬似プロジェクトでTARA/SEC_Concept/SEC_Designドキュメント作成
- [ ] 日本車両セキュリティWG（JASPAR等）勉強会で1回LT登壇

#### 3.2.2 AUTOSAR深掘り
- [ ] DaVinci Configuratorで CRYPTO Stackの全パラメータ表作成
- [ ] Secure Boot → Flash Bootloaderソース解析・フロー図作成

#### 3.2.3 次世代技術準備
- [ ] Rust for Safety-Critical入門（`no_std` + `embedded-hal`でHSMダミードライバ作成）
- [ ] 英語力向上（TOEIC750点 or IELTS 6.0相当）

### 3.3 Phase 3: リード人材（1-3年）

#### 3.3.1 専門性確立
- [ ] "Threat Analysis Leader"として実績積み上げ
- [ ] OEM/ティア1プロジェクトでTARAチームリード（対応策承認率90%目標）

#### 3.3.2 新領域開拓
- [ ] 車載×AIセキュリティ領域参入
- [ ] 生成AI活用「自動脆弱性スキャナ for ECU」PoC社内提案

#### 3.3.3 対外発信
- [ ] 国際カンファレンス発表（escar Europe/Black Hat Asia Automotive Track）

## 4. 焦り・目移り対策

### 4.1 「1週間ルール」
- 新学習テーマ開始後、最低1週間は他に手を出さない
- 1週間後に「続行/中断/方向転換」判断
- 中断時は理由を1行メモ

### 4.2 「小さな成功体験」設計
- 週単位での成果設定
- 毎週金曜「今週できたこと」3つ書き出し
- 小さな成果の社内・SNS共有

### 4.3 AIとの対話活用
- 業務の疑問点、まず5分AIと対話してから先輩に質問
- 質問の質向上によるコミュニケーション効率化

## 5. 最速実行プラン（Vector DaVinci起点）

### 5.1 Step A: 気付きメモ量産（今週〜1ヶ月）
1. DaVinciの各パラメータ・ウィザードで「何のため？」を1行メモ
2. 不明項目はChatGPT/Claudeで解説→自分の言葉でNotion追記
3. 1ヶ月で20-30個メモ蓄積→先輩に10分レビュー依頼

### 5.2 Step B: 社内ミニ勉強会化（1ヶ月〜3ヶ月）
1. メモを10枚スライドに整理（SecOC, CRYPTO Stack概要）
2. 部署内ランチLT（15分）で発表
3. スライド+Q&Aを再整理→新人教育資料化

## 6. 今週の最初の3タスク

### 6.1 即実行タスク
1. [ ] 判断フレーム Step 1-3をNotionで作成（2h）
2. [ ] ISO/SAE 21434章立て写経＋A4 1枚サマリ（3h）
3. [ ] Nucleoボード発注＋mbedTLSビルド環境構築（2h）

### 6.2 今週末集中メニュー（2日間）
#### 土曜日
- [ ] AM: Vector DaVinciの「なぜ？」を1つ調べる
- [ ] PM: ChatGPT/ClaudeでISO21434要点学習（対話形式）

#### 日曜日
- [ ] AM: 同僚・先輩に「車載セキュリティで最重要なこと」ヒアリング
- [ ] PM: 土日の学習内容をA4 1枚にまとめ

### 6.3 1週間デイリータスク
- Day 1: DaVinci CRYPTO設定画面スクショ→不明用語に★マーク
- Day 2: ★用語をAIで質問→Notionに「一言定義」メモ
- Day 3: AESベンチ用STM32 Nucleo発注
- Day 4: ISO21434サマリ執筆（A4 1枚）
- Day 5: 週次レビュー（メモ数/学習時間/疑問点確認）
- Day 6-7: RFC 9147 (EdDSA)をDeepL+AI併用で流し読み→要点Notion化

## 7. 3年後の到達目標

### Year 1: 「質問上手」→「提案できる人」
- 先輩説明の90%理解、的確な質問
- 社内で「○○について調べて」と頼まれるレベル

### Year 2: 「小さなリーダー」
- 新人指導可能
- 顧客要求に技術的代替案提示

### Year 3: 「専門性で頼られる人」
- 部署横断で「セキュリティなら彼/彼女」評価
- 外部セミナー講師レベル

## 8. 重要な確認事項

### 8.1 業務環境
- [ ] 業務時間内「学習メモ」取得の余裕確認
- [ ] 社内LT/勉強会実施許可の文化確認
- [ ] 代替案：オンライン個人ブログ、Qiita Private活用

### 8.2 継続フォロー
- [ ] バックログ完成後のスクリーンショット共有
- [ ] 優先順位・教材の具体化
- [ ] 月次進捗レビューの仕組み構築

## 9. AI時代における車載セキュリティの展望

### 9.1 なぜセキュリティがAI耐性高いか
- セキュリティ = 攻撃者との知恵比べ（完全自動化困難）
- 規制・標準適合に人間の判断・責任必要
- 物理世界との接点で実機検証不可欠

### 9.2 注意点
- ルーチンワーク（設定、文書作成、単純テスト）は確実にAI化
- 「AIを使えないセキュリティエンジニア」は淘汰

### 9.3 目指すべきポジション
**「AIツールを使いこなしながら、物理世界のセキュリティリスクを判断できる人材」**

---

*最終更新: 会話時点*  
*作成者: o3 & Claude Sonnet with Thinking*
