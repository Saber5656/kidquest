# kidquest 設計書 (v1)

- Repository: `github.com/Saber5656/kidquest`
- Tagline: "A family quest board that turns children's chores into game-like tasks."
- License: MIT（前提）/ クラウド送信なし・完全ローカル / 個人 OSS・最小実装で早期リリース
- 作成日: 2026-07-05

---

## 1. コンセプトと既存アプリ・OSS との位置づけ

kidquest は、子どものお手伝い（chore）を「クエスト」に見立てて掲示する家族向けクエストボードである。
モデルは冷蔵庫に貼る紙のお手伝い表・マグネットボード。それを **リビングの iPad や冷蔵庫横のタブレットなど、
家庭の共用デバイス 1 台** に置き換え、リピートの再セットとポイント集計だけを自動化する。
子どもが「できた！」をタップし、親が承認し、貯めたポイントをごほうびに交換する——それだけのアプリを、
**サーバなし・アカウントなし・インストールなし（URL を開くだけ）** で提供する。

**既存の chore アプリ・OSS との位置づけ（車輪の再発明にならない理由）**

| 既存 | 形態 | kidquest との違い |
|---|---|---|
| Privilege Points / Family Rewards 等の商用アプリ | クラウド + 家族アカウント + 親子それぞれのスマホ | kidquest はクラウドもアカウントもなし。**子どもにスマホを持たせない家庭**が対象。データは家の外に出ない |
| donetick / Chorecast（OSS） | Docker 等で自宅サーバに self-host | kidquest はサーバ運用ゼロ。GitHub Pages の URL を開くだけ。self-host 勢が前提にする技術リテラシーを要求しない |
| KidsChores / ChoreOps（Home Assistant 統合） | Home Assistant 環境が前提 | kidquest は HA を持たない一般家庭向け。スマートホーム基盤なしで完結 |
| 紙のお手伝い表・マグネットボード | 物理 | 「冷蔵庫に貼る」気軽さはそのまま、毎日のリセット・ポイント集計・ごほうび台帳を自動化。逆に言えば紙で困っていない家庭には不要（README に正直に書く） |

調査した範囲では、OSS の chore トラッカーはすべて「サーバを立てる」ことが前提であり、
**「共用デバイス 1 台 + 静的 URL だけで動くクエストボード」は空白地帯**。ここが存在理由。
機能の多さで戦わず、**導入の軽さ（1 クリック）とデータがどこにも送られない安心**で差別化する。

---

## 2. v1 スコープ

| 区分 | 項目 | 備考 |
|---|---|---|
| 入れる | 家族メンバー登録（名前 + 絵文字アバター、kid / parent） | ニックネーム推奨。写真は扱わない（§7） |
| 入れる | クエスト作成・編集（タイトル・絵文字・ポイント・毎日 / 毎週(曜日指定) / 単発） | 親エリアで管理 |
| 入れる | 子どもの「できた！」タップ → 親の承認タップ → ポイント加算 | PIN なしの軽い承認（下記判断） |
| 入れる | ごほうび登録とポイント交換 | 「アイス 30pt」「公園 50pt」級のシンプルな台帳 |
| 入れる | 履歴表示（時系列リスト） | 集計グラフは持たない |
| 入れる | JSON エクスポート / インポート | データ保険（§7・§10）。手動バックアップ |
| 入れる | PWA・完全オフライン動作・ホーム画面追加の促し | Service Worker precache。導入体験と永続性の両方に効く |
| 入れる | i18n（ja / en、初期値はブラウザ言語）+ ひらがなモード（ja のみ） | 下記判断 |
| 入れない | **マルチデバイス同期・アカウント・サーバ・プッシュ通知** | **最重要判断**。同期を入れた瞬間にサーバ運用かクラウドが必須になり「クラウド送信なし」原則に違反、規模も爆発する。v1 は単一共用デバイスに絞り、複数デバイスは JSON の手動持ち運びで割り切る |
| 入れない | PIN / パスワードによる親ロック | 共用デバイスは物理的に親の目が届く場所にある。PIN は子どもに見られて即突破される割に UX を悪化させる。承認は不正防止ではなく「親が見たよ」の儀式と割り切る（性善説 + 家庭内運用で十分）。v2 の口だけ残す |
| 入れない | お小遣い・実通貨換算 | 金銭教育アプリはスコープ爆発のもと。ポイントとごほうびまで |
| 入れない | 兄弟ランキング・対抗戦 | 競争を煽る設計は家庭によって逆効果。v1 は各自の達成のみ |
| 入れない | レベル・バッジ・ストリーク等の深いゲーミフィケーション | v1 は絵文字 + 達成演出で十分。反応を見て v2 |
| 入れない | 写真添付（できた証拠写真） | ストレージ膨張 + 子どもの写真というプライバシーリスク（§7） |
| 入れない | カレンダー表示・統計グラフ | 履歴リストで十分。求められたら v2 |

---

## 3. 対応プラットフォームと優先順位

想定デバイスは「家庭の共用デバイス」。置きっぱなしで家族全員が触るものを一次ターゲットにする。

| 優先度 | デバイス / 環境 | 判断 | 理由 |
|---|---|---|---|
| 1 (v1 主戦場) | 共用タブレット（iPad Safari 16+ / Android Chrome）の**ホーム画面追加 PWA** | 対応 | リビング・冷蔵庫横の置きっぱなし運用が本来の用途。フルスクリーン起動 + ストレージ永続性（§7・§10）の両面でホーム画面追加が最適 |
| 2 | 家族 PC のデスクトップブラウザ（Chrome / Edge / Firefox / Safari 最新 2 メジャー） | 対応 | 開発・動作確認も兼ねる。レイアウトはタブレット横持ちと同系 |
| 3 | 親のスマホのブラウザ | 動作はする | ただし**デバイス間同期はしない**。スマホで使うならそのスマホが「唯一のボード」になることを UI で明示 |
| 非対応 | 複数デバイス間のデータ共有・同期 | 明記して見送り | §2 の最重要判断のとおり。README にも "one shared device by design" と書く |
| 非対応 | IE・旧 Android WebView | サポート表明しない | 必要 API（下記）が揃わない |

必要 API: IndexedDB / Service Worker / Web App Manifest / File API（エクスポート・インポート）。いずれも上記ブラウザで安定。

---

## 4. 技術選定

### UI フレームワーク比較

| 候補 | バンドル | 判定 | 理由 |
|---|---|---|---|
| **Preact + TypeScript（採用）** | ~4KB | ✅ | React 互換 API で学習コストなし。共用タブレットの古めの端末でも初回ロードが軽い。作者の既存 OSS（8bitme）とスタックを揃えて保守を一本化 |
| React | ~45KB | ❌ | 本アプリの規模に対して過剰。Preact で足りる |
| Svelte | 小 | ❌ | 品質は高いが、既存プロジェクトとスタックが分かれ個人 OSS の保守負担が増える |
| Vanilla JS | 0 | ❌ | メンバー × クエスト × 承認 × 履歴の状態管理を手書きするとバグの温床。宣言的 UI の恩恵が大きい |

### ローカルストレージ比較

| 候補 | 判定 | 理由 |
|---|---|---|
| **IndexedDB（採用）** | ✅ | 構造化データ・容量余裕・非同期 API。ホーム画面 PWA では ITP の 7 日削除の対象外（§7・§10 の根拠参照） |
| localStorage | ❌ | 5MB 制限と同期 API。履歴が伸びるアプリには不向き。削除ポリシー上の扱いは IndexedDB と同じで利点がない |
| OPFS | ❌ | ファイルシステムが必要な規模ではない。過剰 |

### 採用スタック

| 層 | 技術 | 理由 |
|---|---|---|
| 言語 | TypeScript | ドメインモデル（§5）を型で固定。OSS としてコントリビュートを受けやすい |
| UI | Preact + @preact/signals | 状態は signals で足りる規模。状態管理ライブラリを追加しない |
| ビルド | Vite | GitHub Pages 向け静的出力が容易。8bitme と同一 |
| ストレージ | IndexedDB（`idb` の薄い利用 or 100 行級の自前ラッパ） | Promise 化だけ欲しい。依存は最小に |
| PWA | vite-plugin-pwa | Service Worker 生成の定番。autoUpdate + 更新トースト |
| i18n | 自前の軽量辞書（`t('key')` + ja/en の JSON） | 2 言語固定にライブラリは過剰。ひらがなモードは ja 辞書の別バリアント（`ja-kana`）として実装 |
| テスト | Vitest | ドメインロジック（リピート導出・ポイント計算・日付切替）は純関数にして単体テスト |
| CI/CD | GitHub Actions → GitHub Pages | push → build → deploy。手作業ゼロ |
| 依存ライブラリ | Preact / idb / vite-plugin-pwa 程度 | ゼロには拘らないが最小を保つ |

### 言語ポリシーの判断（日本語家庭 × 英語 OSS の二重性）

- 開発者の家庭 = 日本語、OSS 公開 = 英語 README という二重性がある。**UI は ja / en の 2 言語を最初から持ち、初期値は `navigator.language`、設定でいつでも切替**とする。
- 後から i18n を入れる改修は UI 全面に及ぶため、v1 の最初から辞書引きで書く（キー数はたかが知れている）。
- ひらがなモードは「ja のときだけ現れるサブオプション」とする。en に同等機能（simple words）は作らない——過剰。
- README・コード・コミットは英語。設計書（本書）は日本語。

---

## 5. アーキテクチャ

サーバを持たないため、全体は「UI ⇄ ドメインストア ⇄ IndexedDB」の 3 層 + Service Worker だけで完結する。

```
┌────────────────────────────────────────────────────────┐
│ Browser (installed as home-screen PWA)                 │
│                                                        │
│  ┌────────────────┐  actions        ┌───────────────┐  │
│  │ UI (Preact +    │───────────────▶│ Domain store   │  │
│  │  signals)       │                │ (純 TS)        │  │
│  │  - Board        │◀───────────────│ - 状態遷移      │  │
│  │  - Rewards      │  signals 更新   │ - ポイント計算  │  │
│  │  - History      │                │ - 今日の導出    │  │
│  │  - Parent area  │                └──────┬────────┘  │
│  └────────────────┘                       │ persist    │
│         ▲                                  ▼            │
│         │ install prompt      ┌─────────────────────┐  │
│  ┌──────┴─────────┐          │ Storage adapter      │  │
│  │ Service Worker  │          │ (IndexedDB)          │  │
│  │ (precache 全資産)│          │  ⇄ JSON export/import│  │
│  └────────────────┘          └─────────────────────┘  │
└────────────────────────────────────────────────────────┘
        ネットワーク境界 ═══ 越えるデータなし（§7） ═══
```

- ドメインストアは DOM 非依存の純 TypeScript とし、Vitest で単体テストする（`(state, action) => state`）。
- Storage adapter は「ストア全体のスナップショット + 追記ログ」を IndexedDB に保存する素朴な構成。件数規模（家族 5 人 × 数年分の履歴でも数万件未満）では凝った設計は不要。

### データモデル

```ts
type Member   = { id: string; name: string; emoji: string;
                  role: 'kid' | 'parent'; points: number };
type Quest    = { id: string; title: string; emoji: string; points: number;
                  repeat: 'daily' | 'weekly' | 'once';
                  weekdays?: number[];          // repeat: 'weekly' のみ (0-6)
                  assigneeIds: string[];        // 空 = 全員
                  active: boolean };
type QuestLog = { id: string; questId: string; memberId: string;
                  date: string;                 // 'YYYY-MM-DD'（デバイスローカル）
                  status: 'pending' | 'approved' | 'rejected';
                  completedAt: number; reviewedAt?: number;
                  pointsAwarded?: number };     // 承認時のクエスト点を記録（後から点を変えても履歴不変）
type Reward   = { id: string; title: string; emoji: string;
                  cost: number; active: boolean };
type RedeemLog= { id: string; rewardId: string; memberId: string;
                  redeemedAt: number; cost: number };
type Settings = { locale: 'ja' | 'en'; kanaMode: boolean;
                  onboardingDone: boolean; schemaVersion: number };
```

### 「今日のクエスト」はログから導出する（重要な設計判断）

クエスト自体に「完了フラグ」を持たせて毎日リセットする方式は、**リセット処理を走らせる主体（cron）が
サーバレス構成には存在しない**ため採らない。代わりに:

- 表示時に `Quest × 今日の日付 × QuestLog` から「今日やるべき / 完了済み / 承認待ち」を**毎回導出**する
- `QuestLog.date` にデバイスローカルの日付を焼き込む。日付が変われば導出結果が自然に変わり、リセット処理そのものが不要になる
- 置きっぱなしタブレット（画面を開いたまま日付をまたぐ）対策として、1 分間隔の軽量タイマーと
  `visibilitychange` で日付変更を検出して再描画する
- 日付切替は端末ローカルの 0 時固定（「夜のお手伝いを 25 時扱いにする」等の柔軟性は v1 では持たない）

### 処理フロー（クエスト完了 → 承認 → 交換）

1. 子どもがボードで自分のアバターをタップ → 今日の自分のクエスト一覧
2. 「できた！」タップ → `QuestLog(status: 'pending')` を作成 → 達成演出（§6）。ポイントはまだ動かない
3. 親エリアの承認バッジ（件数表示）から一覧 → 承認 or 差し戻し。承認で `pointsAwarded` を確定し `Member.points` に加算
4. ごほうびショップで交換 → 残高チェック → `RedeemLog` 追記、`points` 減算
5. すべての変更はアクション単位で即 IndexedDB へ書き込み（オートセーブ。保存ボタンは存在しない）

---

## 6. UI/UX

### 画面構成（4 画面 + 親エリア）

```
┌──────────────────────────────────────────────┐
│ kidquest 🏰   [👦れん] [👧ゆい]   ⭐ 120     │ ← 子どもタブ切替 + 残高
├──────────────────────────────────────────────┤
│  きょうのクエスト                              │
│  ┌────────────┐ ┌────────────┐ ┌───────────┐ │
│  │ 🍽 おさら    │ │ 🧸 おもちゃ  │ │ 🛏 ベッド   │ │ ← 大きなカード
│  │ はこび +10  │ │ かたづけ +5 │ │ メイク +5  │ │
│  │ [できた！]  │ │ [できた！]   │ │ ✅ まってね │ │ ← pending は承認待ち表示
│  └────────────┘ └────────────┘ └───────────┘ │
├──────────────────────────────────────────────┤
│  🗡 クエスト | 🎁 ごほうび | 📜 きろく | ⚙(親)  │ ← 下部タブ
└──────────────────────────────────────────────┘
```

- **ボード**: 上記。子どもの操作範囲はここと「ごほうび交換」だけ
- **ごほうびショップ**: カードに交換可否を色で表示（足りないカードは灰色 + 「あと 12pt」）
- **きろく**: 「7/5 れん 🍽 おさらはこび +10」の時系列リスト
- **親エリア（⚙）**: メンバー / クエスト / ごほうびの CRUD、承認一覧、JSON エクスポート・インポート、言語設定。削除・編集系の操作はすべてここに集約

### 子ども向け UI の設計原則

| 原則 | 実装 |
|---|---|
| タップターゲットは大きく | カード・ボタンは最小 64px 角。ボードはタブレット横持ちで 1 画面に収まる量だけ表示 |
| 文字より絵文字 | クエスト・ごほうび・アバターは絵文字必須（画像アセットを持たない設計でもある） |
| 読めない子への配慮 | ひらがなモード（漢字を含む UI 文言を全ひらがな辞書に差し替え）。未就学児はアイコンとポイント数字だけでも回る構成 |
| 達成感の演出 | 「できた！」で紙吹雪 + アバターがジャンプするマイクロアニメ + ポイントのカウントアップ。音は既定 OFF（リビング設置で音は事故のもと） |
| 壊せない UI | 子どもが触れる範囲に削除・編集・設定を置かない。誤タップの取り消しは「できた！」直後 5 秒間だけ undo を表示 |
| 兄弟の分離はゆるく | アバタータブの切替だけで本人確認はしない（性善説）。他人のクエストを押してしまっても親の承認段階で気づける |

### 初回起動体験

1. URL を開くと**サンプル家族（👦 + 👧 + 🧑 パパ）とサンプルクエストでボードが最初から動いている**。まず触って理解してもらう
2. 「じぶんの家族にする」ボタン → サンプルを消してメンバー登録（名前 + 絵文字を選ぶだけ、2 分）
3. 登録完了時に**ホーム画面追加の案内**を表示。iOS Safari は共有シート → 「ホーム画面に追加」の手順を図解（beforeinstallprompt が使える Android Chrome はネイティブプロンプトを発火）。「追加するとデータが消えにくくなり、アプリみたいに開けます」と理由も添える（§7・§10 の永続性保険）
4. 以後の起動はホーム画面アイコンからフルスクリーン。チュートリアルはこれで終わり

### エッジケースの扱い（v1）

- 承認待ちが溜まる → 親エリアのタブにバッジ件数。放置されたら翌日以降も pending のまま残す（自動承認はしない）
- ポイント不足で交換タップ → 「あと Npt」を表示して励ます文言。エラー扱いにしない
- 日付をまたいで画面が開きっぱなし → §5 のタイマー + visibilitychange で自動更新
- インポートで既存データがある → 「上書きされます」の明示確認 + 直前データの自動エクスポートを挟む
- ブラウザデータ消去・端末変更 → JSON バックアップから復元（README・親エリアに手順明記）

---

## 7. プライバシー設計

子どもの名前・行動記録という**児童の個人情報**を扱う自覚を持ち、独立節として設計する。

### 原則

1. **データを外部に送る経路をそもそも作らない**（バックエンド・API・外部 SaaS が存在しない）
2. **設定で OFF ではなく、能力として不可能に**（構造的担保）
3. **収集しないことで規制対応を構造的に完了させる**

### 担保策

| 層 | 施策 |
|---|---|
| アーキテクチャ | 全データは端末内 IndexedDB のみ。ネットワークを流れるのは GitHub Pages からの静的資産 DL だけで、上りのデータ送信は一切ない |
| CSP | `connect-src 'self'`、`img-src 'self' data:` を宣言。外部へ fetch する能力をブラウザレベルで封じる |
| 計測ゼロ | アナリティクス・トラッカー・外部フォント・CDN を一切使わない |
| オフライン実証 | 初回ロード後は機内モードで全機能が動く（PWA）。「機内モードで試して」を README に明記——非開発者向けの最強の証明 |
| 検証手順の公開 | README に「DevTools → Network タブを開いて操作 → リクエストが飛ばないことを確認」の手順を掲載 |
| OSS | 全コード公開。ビルドは GitHub Actions 上で行われ、配信物とソースの対応を検証できる |

### COPPA / GDPR-K 的整理

- COPPA（米）や GDPR の児童データ規定が課すのは「事業者が児童データを**収集・処理**する」場合の義務。
  kidquest は**運営者にデータが一切渡らない**（そもそも受け取るサーバがない）ため、収集自体が発生せず、
  同意取得・保護者確認・削除請求といった義務の前提を構造的に欠く。この整理を README の Privacy 節に明記する
- とはいえ配慮は上乗せする: **名前はニックネームや呼び名を推奨**（UI プレースホルダを「れん」「ゆうくん」等にする）、
  **写真・生年月日・位置情報は仕様として扱わない**（§2 で「入れない」判断済み）
- エクスポート JSON は子どもの記録を含むファイルである旨を出力時に一言表示（「家族の外に共有しないでね」）

### データの寿命と保険（永続性はプライバシーの裏面）

- 根拠: WebKit の ITP は Safari ブラウザ内のサイトデータ（IndexedDB 含む script-writable storage）を
  「7 日間ユーザー操作がなければ削除」するが、**ホーム画面に追加した web app はこの 7 日 cap の対象外**
  と WebKit 公式ブログが明言している（参考資料）。よって一次対策は「ホーム画面追加を促す」（§6 初回体験）
- 二次対策として **JSON エクスポート / インポート**を v1 に含める。端末の故障・買い替え・誤消去への保険であり、
  同期を持たない本設計における唯一のデータ移動手段でもある
- 対応環境では `navigator.storage.persist()` も呼ぶ（害はない）。ただし Safari での実効性は保証を確認できて
  いないため、頼りにせず #1 spike の観測項目とする（§10）

---

## 8. 配布方法

| 項目 | 内容 |
|---|---|
| ホスティング | GitHub Pages（`https://saber5656.github.io/kidquest/`）。独自ドメインは後回し |
| デプロイ | GitHub Actions: main への push → Vite build → Pages deploy。手作業ゼロ |
| インストール | なし。URL を開く → ホーム画面に追加（推奨）。ストア申請はしない |
| 更新 | 静的アプリのため再訪で最新。vite-plugin-pwa の autoUpdate + 「新しいバージョンがあります」トースト |
| バージョニング | タグ + GitHub Releases に CHANGELOG。データは `schemaVersion` を持たせ、旧エクスポート JSON の取込にマイグレーションを用意 |
| ライセンス | MIT。絵文字ベースのためアセットのライセンス管理は不要 |

---

## 9. README 構成案（英語）

```markdown
<バナー画像: クエストボード風ロゴの横長 PNG（絵文字ベースの軽いもの）>

# kidquest 🏰
> A family quest board that turns children's chores into game-like tasks.

<デモ GIF: 子どもが「できた！」→ 紙吹雪 → 親が承認 → ポイント加算 → ごほうび交換の 12 秒ループ>

**👉 Open the board: https://saber5656.github.io/kidquest/** — no install, no account, no cloud.

## What it is
- A chore board for **one shared family device** (the living-room iPad, the fridge tablet)
- Kids tap "Done!", parents tap "Approve", points add up, rewards get redeemed
- Works fully offline (PWA). Add it to the home screen and it feels like an app

## Privacy — your family's data never leaves the device
1. There is no server. Nothing is uploaded, ever.
2. Load it once, turn on Airplane Mode — everything still works.
3. Open DevTools → Network while using it: zero requests.
4. Back up anytime with JSON export/import.

## One device by design
- 同期・アカウントを持たない設計判断を 3 行で説明（"If you want multi-device sync,
  this is not that tool" と正直に書き、self-host 系 OSS へのリンクを添える）

## Getting started / For developers（clone / npm i / npm run dev の 3 行）
## License
MIT
```

ポイント: デモ GIF と「Open the board」の 1 行導線がファーストビューに収まること。
バッジは license / deploy (Pages) / PRs welcome の 3 つまで。

---

## 10. リスクと実装前検証項目

| 優先度 | 項目 | 内容 | 検証方法 |
|---|---|---|---|
| P0 | **ホーム画面 PWA での IndexedDB 長期永続性** | WebKit は「ホーム画面 web app は 7 日削除の対象外」と公言しているが、実機・現行 iOS での長期残存と `navigator.storage.persist()` の挙動は自分で確かめる。ここが崩れると本設計のデータ基盤が崩れる | 最小 PWA を Pages に deploy し、iPad 実機でホーム画面追加 → IndexedDB 書込 → **放置観測を開発と並行して走らせる**（Issue #1）。ブラウザタブ運用（非追加）との差も記録 |
| P1 | 日付導出ロジックのバグ | 「今日の分」導出・週次曜日・undo・承認の組合せでポイント二重加算や取りこぼしが起きやすい | ドメインストアを純関数にして Vitest で日付固定テスト（日付またぎ・週またぎ・タイムゾーン変更） |
| P1 | 子どもが承認フローを理解できるか | 「できた！のあとに親の承認がある」の 2 段階が未就学児に伝わるか。pending 表示の文言・見た目で決まる | 自分の家庭でユーザーテスト（対象年齢の子で「できた！→ まってね表示」が混乱しないか観察）。文言を差し替えられる構造にしておく |
| P2 | インポートの上書き事故 | 子どもの数ヶ月分の記録を親が誤って消す事故は信頼を失う | インポート直前の自動エクスポート実装 + 確認ダイアログの文言レビュー |
| P2 | 古い共用タブレットでの動作 | 家庭の「余ったタブレット」は OS が古いことが多い | 対応下限（iOS Safari 16+ / Android Chrome）を README に明記し、下限実機 or BrowserStack で smoke test |
| P2 | PWA キャッシュの更新不全 | 旧バージョンが残り続ける | vite-plugin-pwa の autoUpdate + 更新トースト（8bitme と同じ既知の対処） |
| P3 | 名前空間の衝突 | "kidquest" 類似名の既存プロダクト・商標の簡易確認 | リリース前に検索 |

**最重要リスク**: P0 の永続性。WebKit の公式声明という根拠はあるが、家族の数ヶ月分の記録を預かる以上、
実機での裏取りなしに出荷しない。なお放置観測は日数を要するため、**#1 で観測を最初に開始して
開発と並行させる**（結論を待ってから作り始めるのではなく、二重保険の JSON エクスポートを №6 で必ず実装することで
どちらに転んでも v1 が成立する構成にしてある）。

---

## 11. v1 Issue 分割案（8 個）

- **#1 `Spike: verify IndexedDB persistence of a home-screen PWA on iOS/Android`** — ラベル: `spike`, `design`
  最小 PWA（IndexedDB へ書込 + 書込日時表示 + `navigator.storage.persist()` 呼出と結果表示のみ）を Pages に deploy し、iPad 実機でホーム画面追加運用とブラウザタブ運用の両方で放置観測を開始する。P0 リスク（7 日削除の適用範囲）を実機で裏取りする。
  受け入れ条件: 観測用 PWA が deploy され、両運用で 7 日超の放置後もホーム画面側のデータが残ることを確認し、`persist()` の返り値と挙動を含む観測記録が Issue コメントに残っている。

- **#2 `Set up project skeleton: Vite + Preact + TypeScript + PWA + Pages deploy`** — ラベル: `infra`
  Vite + Preact + TypeScript の雛形、vite-plugin-pwa（precache + autoUpdate + 更新トースト）、CSP メタタグ、GitHub Actions → GitHub Pages の自動デプロイを設定する。
  受け入れ条件: main への push で Pages に deploy され、初回ロード後に機内モードでページが開き、DevTools Network で外部リクエストが 0 件である。

- **#3 `Implement domain model, today-derivation logic, and IndexedDB persistence`** — ラベル: `enhancement`
  §5 のデータモデル、「今日のクエスト」導出（daily / weekly / once、日付またぎ検出）、ポイント計算、pending → approved の状態遷移、IndexedDB への保存・読込を DOM 非依存の純 TS で実装する。
  受け入れ条件: 日付またぎ・週またぎ・二重加算防止を含む Vitest が通り、リロード後も状態が復元される。

- **#4 `Build quest board screen with kid-first interactions`** — ラベル: `enhancement`, `ux`
  ボード画面（アバタータブ切替・今日のクエストカード・「できた！」→ 紙吹雪 + カウントアップ演出・5 秒 undo・承認待ち表示)を実装する。タップターゲット 64px+、タブレット横持ち最適化。
  受け入れ条件: iPad 実機で子どもの一連の操作（選ぶ → できた！→ まってね表示）が完結し、演出が 60fps 近辺で動作する。

- **#5 `Implement approval flow, reward shop, and history`** — ラベル: `enhancement`
  親エリアの承認一覧（バッジ件数付き）、承認 / 差し戻し、ごほうび CRUD とポイント交換（残高チェック・「あと Npt」表示）、履歴の時系列リストを実装する。
  受け入れ条件: 承認でのみポイントが加算され、残高不足の交換が防止され、全操作が履歴に残る。

- **#6 `Add family setup, parent area, and JSON export/import`** — ラベル: `enhancement`
  メンバー・クエストの CRUD（親エリア集約）、サンプルデータからの初期化、JSON エクスポート / インポート（schemaVersion、インポート直前の自動バックアップ、上書き確認）を実装する。
  受け入れ条件: サンプル → 自分の家族への移行が UI だけで完結し、エクスポート → 全消去 → インポートで完全に復元される。

- **#7 `Add i18n (ja/en), hiragana mode, and first-run experience`** — ラベル: `enhancement`, `ux`
  軽量辞書による ja / en 切替（初期値はブラウザ言語）、ひらがなモード（ja のみ）、初回起動フロー（サンプルボード → 家族登録 → ホーム画面追加案内の図解）を実装する。
  受け入れ条件: 全 UI 文言が辞書経由で、言語・ひらがな切替が即時反映され、初回フローが iOS / Android 両方の手順案内を表示する。

- **#8 `Write English README with banner, demo GIF, and one-click URL`** — ラベル: `docs`
  §9 の構成で README を作成する。バナー・12 秒デモ GIF・「Open the board」導線・プライバシー検証手順（機内モード / Network タブ）・"one device by design" の説明を含める。
  受け入れ条件: デモ GIF と URL 導線がファーストビューに収まり、バッジが 3 個以内で、プライバシー検証手順が再現可能である。

推奨着手順: #1（観測開始だけ先行）→ #2 → #3 → #4 → #5 → #6 → #7 → #8。
#1 の放置観測は #2〜#6 と並行して進み、v1 リリース判断までに結論を出す。

---

## 参考資料（設計時の調査ソース）

- iOS の 7 日削除とホーム画面 web app の除外（一次ソース）: [Full Third-Party Cookie Blocking and More — WebKit Blog](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/) / [Tracking Prevention in WebKit](https://webkit.org/tracking-prevention/)
- 同・補足情報: [Safari iOS PWA Data Persistence Beyond 7 Days — Apple Developer Forums](https://developer.apple.com/forums/thread/710157) / [What Safari's 7-day cap on script-writeable storage means for PWA developers — Search Engine Land](https://searchengineland.com/what-safaris-7-day-cap-on-script-writeable-storage-means-for-pwa-developers-332519) / [PWA iOS Limitations and Safari Support — MagicBell](https://www.magicbell.com/blog/pwa-ios-limitations-safari-support-complete-guide)
- 類似 OSS（サーバ / self-host 前提であることの確認）: [donetick/donetick](https://github.com/donetick/donetick) / [ad-ha/kidschores-ha（Home Assistant 統合）](https://github.com/ad-ha/kidschores-ha) / [ccpk1/choreops](https://github.com/ccpk1/choreops) / [KenWeTech/Chorecast](https://github.com/KenWeTech/Chorecast) / [GitHub topic: chores](https://github.com/topics/chores)
- 類似商用アプリ（クラウド + マルチデバイス前提であることの確認）: [Privilege Points](https://privilegepoints.com/) / [Family Rewards](https://www.familyrewards.app/) / [Homechart](https://homechart.app/)

---

## Changelog

- 2026-07-05: 初版
