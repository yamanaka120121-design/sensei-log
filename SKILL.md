---
name: sensei-log-calendar
description: SENSEI.LOG パーソナルカレンダー＆ライフログアプリの設計・実装スキル。山中優弥（高校教員）専用。`/Users/yamanakayuuya/my-calendar/index.html` の修正・機能追加・継続開発を行う際に使用する。アーキテクチャ、デザインシステム、実装済み機能、データモデル、開発フェーズを含む。
---

# SENSEI.LOG カレンダー開発スキル

## プロジェクト概要

**ファイル:** `/Users/yamanakayuuya/my-calendar/index.html`（単一ファイルSPA）  
**公開URL:** `https://yamanaka120121-design.github.io/sensei-log/`（GitHub Pages）  
**フェーズ:** Phase 1（HTMLモックアップ）完了 → Phase 2（Next.js + Supabase）が次のステップ  
**コンセプト:** 「静寂と内省の黒の秘書」——教員の公私の時間を整理し、日々のログから自己複製AIを育てる

関連資料:
- [仕様書](spec.md)
- [スマホ画面設計](mobile-design.md)
- `my-calendar/my-life-philosophy.md` — ユーザーの価値観・思考パターン（朝の問い・今日の一言の元ネタ）
- `my-calendar/music-philosophy.md` — 吹奏楽部あり方10条（将来のLINE連携で使用）
- `my-calendar/school-schedule.md` — 学校時間割バリアント定義
- `my-calendar/my_lifestyle_review_sheet.md` — 習慣チェック24項目の元となるシート

---

## デザインシステム

```css
/* カラーパレット */
--bg: #080c14          /* 宇宙紺・背景 */
--bg-col: #0d1320      /* カラム背景 */
--text: #e2e8f0        /* メインテキスト */
--text-muted: #64748b  /* ミュートテキスト */
--text-dim: #374151    /* 最も薄いテキスト */
--border: #1e293b      /* ボーダー */

/* カテゴリカラー */
--green:  #4ade80   /* 授業 */
--cyan:   #00c8e0   /* 部活動 */
--red:    #f87171   /* 学校行事 */
--purple: #a78bfa   /* プライベート */
--gray:   #6b7280   /* その他 */
--amber:  #fbbf24   /* 習慣50%スコア用 */

/* フォント */
--font-jp:   'Noto Sans JP'    /* 日本語UI */
--font-mono: 'JetBrains Mono'  /* 数値・コード */
```

**時間軸:** `START_HOUR = 0`、`END_HOUR = 24`、`PX_PER_MIN = 1.5px`  
（`TOTAL_H = 24 × 60 × 1.5 = 2160px`）

---

## アーキテクチャ

### 主要状態変数

```javascript
let weekOffset      = 0        // 今週から何週ずれているか
let selectedDay     = N        // 0=日〜6=土（日表示・スマホで使用）。週末は1（月）にデフォルト
let mobSelectedDay  = N        // モバイル週ストリップの選択日（selectedDay と同期）
let currentView     = 'week'   // 'week' | 'day' | 'month'
let editingEventId  = null     // モーダル編集中のイベントID
let userEvents      = []       // localStorage から読み込み
let dayPatterns     = {}       // 日付ごとの時間割パターン
let habitDate       = 'YYYY-MM-DD' // 習慣チェックの表示日
```

### 重要な関数

| 関数 | 役割 |
|------|------|
| `buildCalendar()` | カレンダー全体を再描画 |
| `getEventsForWeek(wOffset)` | DEFAULT_EVENTS＋userEvents をマージして返す |
| `getEventsForDate(dateStr)` | 指定日のイベント一覧（月ビュー・週ストリップ用） |
| `buildSchoolSchedule({asagaku, periodMins})` | 時間割バリアント生成 |
| `buildMobWeekStrip()` | モバイル週間ストリップを描画 |
| `buildMonthViewDesktop()` / `buildMonthViewMobile()` | 月表示を描画 |
| `renderHabitItems(container, dateKey, onChange)` | 習慣チェック項目を描画（binary/scale 混在） |
| `calcHabitScore(dateData)` | `{answered, total, avgPct}` を返す（binary/scale 統合計算） |
| `renderHabitTab()` | 習慣タブ全体を再描画 |
| `initGreeting()` | 今日の一言オーバーレイ（モバイル・1日1回） |
| `showHabitCheckOverlay(today)` | 強制習慣チェックオーバーレイ |
| `switchMobTab(tab)` | 'cal' | 'log' | 'habit' | 'month' でタブ切替 |
| `triggerAIReview(text, prefix)` | 振り返り送信後にAI添削モックを表示（約1.4秒遅延） |
| `shift(dir)` | PC週表示はweek単位、PC日表示・スマホはday単位でナビゲート |
| `openModal(eventId, date, start, end)` | 予定追加/編集モーダルを開く |

### PCレイアウト

```
header（52px）
├── サイドバー（192px）: カテゴリ凡例 / ミニカレンダー / 週統計
├── カレンダーメイン（flex:1）: 週表示 / 日表示 / 月表示
└── 右パネル（272px）: 朝の問い / 振り返り入力 / AI添削 / 過去ログ（折りたたみ）
```

### モバイルレイアウト

```
header（44px）: SENSEI.LOG / 日付 / ＋ボタン
├── 週間ストリップ（タブがcalのとき表示）
├── cal-scroll（カレンダー本体）
│
├── [固定・右縦ピル] mobile-nav: カレンダー / ログ / チェック / 月
│
├── mob-log（fixed・logタブ時）: 朝の問い / 今日の振り返り / 昨日の振り返り
├── mob-habit（fixed・habitタブ時）: 習慣チェック24項目
├── mob-month（fixed・monthタブ時）: 月カレンダー
├── greeting-overlay（fixed・z:500）: 今日の一言
└── habit-overlay（fixed・z:510）: 強制習慣チェック
```

---

## データモデル（localStorage）

```javascript
// イベント（ユーザー追加・特定日付）
{ id: 'uid', title: '...', cat: 'lesson|club|event|private|other',
  date: 'YYYY-MM-DD', start: 'HH:MM', end: 'HH:MM', sub: '...' }

// 振り返りログ
{ date: 'YYYY-MM-DD', body: '150字以内のテキスト' }

// 日付ごとの時間割パターン
{ 'YYYY-MM-DD': 'std' | 'no_ag' | 'std45' | 'no_ag45' | 'none' }

// 習慣チェックデータ
{ 'YYYY-MM-DD': { 'habit_id': 0|1|2|...|5 } }
// binary: 0=未回答, 2=できた, 1=できなかった
// scale:  0=未回答, 5=100%, 4=80%, 3=50%, 2=20%, 1=✕
```

---

## 学校時間割システム

`buildSchoolSchedule({ asagaku=true, periodMins=50 })` で生成。4バリアント：

```javascript
SCHOOL                 // 標準（朝学あり・50分）
SCHOOL_NO_ASAGAKU      // 朝学なし・50分
SCHOOL_45              // 朝学あり・45分
SCHOOL_NO_ASAGAKU_45   // 朝学なし・45分
```

PATTERNS オブジェクト: `{std, no_ag, std45, no_ag45, none}` でパターンを保持。  
`getDayPattern(dateStr)` で指定日のパターンを取得（デフォルト: 'std'）。

---

## 習慣チェックシステム詳細

### HABITS 配列の構造

```javascript
// binary型
{ id: 'bath', type: 'binary', label: '湯船に入った',
  cat: '🌙 睡眠・生活リズム', def: '39〜40℃の湯船に10〜15分、肩までつかる' }

// scale型
{ id: 'posture', type: 'scale', label: '仕事中に良い姿勢を保てた',
  cat: '💪 運動・姿勢', def: '仙骨・肩甲骨・後頭部の3点が壁に触れるような姿勢',
  criteria: {5:'終日ほぼ完璧に意識できた', 4:'...', 3:'...', 2:'...', 1:'...'} }
```

### calcHabitScore(dateData) の動作

```javascript
// binary: できた(val=2)=100点、できなかった(val=1)=0点
// scale:  5=100, 4=80, 3=50, 2=20, 1=0
// 未回答(val=0)は平均計算から除外
// 戻り値: { answered, total, avgPct }
```

---

## 今日の一言（greeting）システム

`DAILY_MOMENTS[曜日]` オブジェクト（7要素）：
```javascript
{ msg: '一言メッセージ', sub: 'サブテキスト',
  q: '今日の問い', reflect: '夕方に振り返ること' }
```

`MORNING_QUESTIONS[曜日]`（7要素）：朝の問い（right-panelにも表示）

---

## 実装上の注意点

- `toY('HH:MM')` は START_HOUR(0) 基準のピクセル値を返す。時刻文字列は `'08:30'` 形式で
- `shift()` は `isMobile() || currentView === 'day'` で日単位ナビ、それ以外は週単位
- `selectedDay` は 0〜6（0=日曜）。週末（土日）は月曜（1）にデフォルト
- `col.dataset.date` に日付文字列をセット → 週ストリップのタップで mob-show を切替
- `body.view-day` クラスでPC日表示の列非表示を制御（CSSのみで動作）
- `body.view-month` クラスで月表示（cal-scroll を非表示、month-view-container を表示）
- `triggerAIReview(text, prefix)` の prefix は desktop=`''`、mobile=`'mob-'`
- スキーマバージョン `sensei_schema_v2` が localStorage にない場合のみ初回クリア実行
- プリセット: `sensei_preset_20260519`（初回投入）・`sensei_preset_20260519v2`（修正パッチ）
- isMobile() は `window.innerWidth < 768` で判定
