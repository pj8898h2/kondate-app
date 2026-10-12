# kondate-app MVP 設計仕様

- 作成日: 2026-10-12
- 状態: レビュー待ち
- 対象: Slice 1(MVP)。Slice 2 は範囲だけを示す

## 1. 目的

週に1回まとめ買いをする2人世帯が、**これまで食べた料理への評価をもとに、今週の夕食の献立を数タップで決められる**ようにする。

あわせて、このリポジトリでは GitHub を軸にした開発の進め方(設計ドキュメント → Issue → PR → CI/CD、テスト駆動開発、AI エージェントによる仕様駆動開発)を試す。

### 成功の基準(Slice 1)

- 4週連続で、その週の献立をこのアプリで決めた
- 確定した献立の夕食のうち 7 割以上に評価が付いた(入力が続いていることの確認)

### 最大のリスクと方針

最大のリスクは**入力が続かないこと**。すべての画面で「1回の操作で入力する量を最小にする」ことを優先する。

- 評価は食べた後に、美味しさを1タップで付けるだけにする
- 「食べた」という記録は、献立を確定した時点で自動で作る
- 初期データは一括で投入し、使い始めた日から献立を組めるようにする

## 2. 利用者と前提

| 項目 | 内容 |
|---|---|
| 利用者 | 2人世帯。評価は世帯で1つにする(2人の評価を区別しない) |
| 端末 | スマホのブラウザ。ホーム画面に追加して使う(PWA) |
| 共有 | Slice 1 では1人で使う。Slice 2 で同じパスワードを共有する |
| 対象の食事 | 夕食のみ。1日1品 |

## 3. スコープ

### Slice 1(MVP)

- 料理の登録・編集・一覧
- 初期データの一括投入スクリプト
- 献立の生成(開始日・日数・今週の気分を指定)
- 日ごとの差し替え、献立の確定
- 確定した献立への評価(美味しさを5段階)
- パスワード1つによるアクセス制限
- 本番環境への公開、CI、PWA の manifest

### Slice 2(範囲のみ)

買い物リスト、配偶者との共有、OCI への移行、外部レシピの URL 取り込み、LLM による献立の説明と並べ替え、冷蔵庫の余り物からの提案

### 対象外

ネイティブアプリ、外部サイトのスクレイピング、主菜と副菜の組み合わせ、オフライン対応、複数の世帯への対応

## 4. アーキテクチャ

**Next.js(App Router, TypeScript)+ Drizzle ORM + PostgreSQL(Supabase)+ Vercel**

```
src/
  domain/menu/     献立生成の純粋関数(UI と DB に依存しない)
  db/              Drizzle のスキーマとリポジトリ層(DB アクセスはここに閉じ込める)
  app/             画面と Server Actions
  middleware.ts    パスワードによるアクセス制限
scripts/seed.ts    初期データの一括投入
seed/              投入するデータ(sample.csv のみ Git に入れる)
```

### 設計判断(詳細は ADR に記録する)

| 判断 | 理由 |
|---|---|
| Supabase は素の Postgres として使い、Drizzle で接続する | Slice 2 で OCI の VM 上の Postgres に移すとき、接続文字列を変えるだけで済むようにする |
| 認証はパスワード1つ(環境変数に置き、middleware で確認して Cookie に保存) | 評価が世帯で1つなので、ユーザーを区別する必要がない。共有するときはパスワードを渡すだけで済む |
| 推薦はルールによる点数付けで行う | 結果を毎回同じにでき、テストを先に書ける。LLM は Slice 2 で説明と並べ替えに使う |
| OCI の Autonomous Database は使わない | Drizzle が Oracle に対応していない。Always Free では7日間使わないと自動で停止する |

## 5. データモデル

```ts
dishes {
  id: serial PK
  name: text NOT NULL
  effort: int NOT NULL CHECK (1..5)   // 1=らく, 5=大変
  sourceUrl: text NULL
  memo: text NULL
  createdAt: timestamptz DEFAULT now()
}

ingredients {
  id: serial PK
  dishId: int FK -> dishes.id ON DELETE CASCADE
  name: text NOT NULL
  isMain: boolean NOT NULL DEFAULT false   // 1つの料理につき最大1つ
}

meals {
  id: serial PK
  date: date NOT NULL UNIQUE               // 1日1品
  dishId: int FK -> dishes.id
  taste: int NULL CHECK (1..5)             // 食後に付ける。未評価は NULL
}
```

- **料理の美味しさ** = その料理の `meals.taste` の平均。評価が1件もない料理は **3** として扱う
- **主食材**は `isMain` の材料の名前とする。「今週の気分」の選択肢は、登録されている主食材の名前を重複なしで並べたもの
- 手間は料理の属性で、登録や編集のときに付ける(食べるたびには付けない)

## 6. 献立生成ロジック

`src/domain/menu/` に純粋関数として作る。

```ts
type MenuOptions = { startDate: string /* YYYY-MM-DD */; days: number /* 1..7 */; moods: string[] }
type MenuDay = { date: string; dishId: number; score: number }

generateMenu(dishes: DishWithStats[], meals: Meal[], options: MenuOptions): MenuDay[]
rerollDay(menu: MenuDay[], dayIndex: number, dishes: DishWithStats[], meals: Meal[], options: MenuOptions): MenuDay[]
```

### 点数(料理 d を日付 t に置くとき)

```
score(d, t) = taste(d)                     // 1..5、評価なしは 3
            - weekdayPenalty(d, t)         // t が月〜金 かつ effort(d) >= 4 なら 2、それ以外は 0
            - recencyPenalty(d, t)         // d を最後に食べた日から t までの日数を n とする。n <= 14 なら 3 * (15 - n) / 14、それ以外は 0
            + moodBonus(d)                 // d の主食材が moods に含まれていれば 1.5、それ以外は 0
```

- 「最後に食べた日」は、`meals` のうち t より前の日付で最も新しいものとする
- 気分は**加点**として扱い、候補を絞り込む条件にはしない(該当する料理が少ない週でも献立が組めるように)
- 係数は `src/domain/menu/weights.ts` に定数としてまとめ、後で調整できるようにする

### 選び方

- 開始日から1日ずつ、その日の点数が最も高い料理を選ぶ(貪欲法)
- **同じ献立の中で同じ料理は出さない**
- 同点のときは `dishId` の小さい方を選ぶ(結果を毎回同じにするため)
- 料理の数が日数より少ないときは、作れた日数分だけを返す(エラーにはしない)
- `rerollDay` は、その日について「現在の料理と、献立の他の日に入っている料理を除いた中で」点数が最も高いものに入れ替える。候補がなければ元のまま返す

### 確定

確定すると、各日の `meals` を upsert する(同じ日付にすでに記録があれば料理を置き換え、評価は NULL に戻す)。

## 7. 画面

| 画面 | パス | 内容 |
|---|---|---|
| 今週 | `/` | 今日から7日分の `meals` を表示する。食べた日は美味しさを ★1〜5 で1タップで付ける。料理名をタップすると別の料理に差し替えられる |
| 献立を作る | `/plan` | 開始日(既定は次の土曜日)、日数(既定は7)、気分(主食材のチップ、複数選択可)→ 生成 → 各日の「別のにする」→ 確定 |
| 料理 | `/dishes` | 一覧(名前・手間・美味しさの平均・最後に食べた日)、登録・編集(名前、手間、材料と主食材の印、URL、メモ) |
| ログイン | `/login` | パスワードを1つ入力する |

## 8. エラー処理

- パスワードが違うときは `/login` にメッセージを出す。回数制限は Slice 1 では付けない
- 入力のチェックは Server Actions で行う(effort と taste が 1〜5 の範囲か、名前が空でないか)。違反していれば画面にメッセージを出す
- 料理の登録数が日数より少ないときは、作れた日数分の献立を出し、「料理が足りません」と表示する
- DB に接続できないときは、Next.js の error boundary で再読み込みを促す

## 9. テスト

- **単体テスト(Vitest、先に書く)**: `generateMenu` と `rerollDay`
  - 平日に手間 4 以上の料理は減点される / 休日は減点されない
  - 14日以内に食べた料理は減点され、日が近いほど減点が大きい
  - 気分に合う主食材の料理は加点される。気分に合う料理が足りなくても献立は組まれる
  - 同じ献立の中で料理が重複しない
  - 同点のときは dishId の小さい方が選ばれ、同じ入力なら同じ結果になる
  - 評価のない料理は美味しさ 3 として扱われる
  - 料理が日数より少ないときは、作れた日数分だけが返る
  - `rerollDay` は他の日の料理と重複しない料理に入れ替え、候補がなければ元のまま返す
- **CI**: GitHub Actions で PR ごとに `lint` / `typecheck` / `test` を回す
- **手動の受け入れ確認**: iPhone でホーム画面に追加 → 7日分の献立を作る → 1日差し替える → 確定する → 翌日に評価を付ける → 翌週の献立で、その料理が減点されていることを確かめる

## 10. 開発の進め方

1. この仕様書と ADR(`docs/adr/`)をレビューし、確定させる
2. 仕様書から Issue を起こし、Milestone を「Slice 1」と「Slice 2」の2つ作る
3. 1 Issue = 1 ブランチ = 1 PR。Issue と仕様書を AI エージェント(Claude Code)に渡して実装させ、人間がレビューしてマージする
4. Vercel の GitHub 連携で、PR ごとにプレビューを出し、main へのマージで本番に出す

## 11. Public リポジトリとしての注意

- `.env*` は Git に入れない。秘密の値は Vercel の環境変数に置く
- 家庭の料理データ(`seed/*.csv`)は Git に入れない。入れるのは `seed/sample.csv` だけにする
