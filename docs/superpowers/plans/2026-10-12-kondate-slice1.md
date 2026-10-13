# kondate-app Slice 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 料理の登録・評価から、日〜金などの夕食献立をルールで生成・差し替え・確定し、スマホのホーム画面から使える状態で本番公開する。

**Architecture:** 献立ロジックは `src/domain/` の純粋関数(TDD で作る)。DB アクセスは `src/db/repo.ts` に閉じ込め、Drizzle + postgres-js で Supabase の Postgres に素の Postgres として接続する。画面は Next.js 16 App Router の Server Components と Server Actions、アクセス制限は `src/proxy.ts` と単一パスワード。

**Tech Stack:** Node 22 / Next.js 16 (App Router, TypeScript, Tailwind) / Drizzle ORM + drizzle-kit / postgres (postgres-js) / zod 4 / csv-parse / Vitest / GitHub Actions / Vercel / Supabase

**Spec:** `docs/superpowers/specs/2026-10-12-kondate-mvp-design.md`

## Global Constraints

- Node 22。`package.json` の `engines` は付けない(Vercel の既定に合わせる)
- Next.js 16: アクセス制限のファイルは **`src/proxy.ts`(関数名 `proxy`)**。`middleware.ts` は使わない(16 で非推奨)
- 日付はすべて `'YYYY-MM-DD'` 文字列で扱う。「今日」は **Asia/Tokyo** で求める(Vercel のサーバーは UTC)。日付の計算は UTC の 0 時として行う
- 点数の係数(spec §6 の値を変えない): 平日の重い手間 = effort ≥ 4 で −2 / 最近食べた = 14日以内で `3 * (15 - n) / 14` / 気分 = +1.5 / 評価がない料理の美味しさ = 3
- 同点のときは dishId の小さい方。同じ入力なら同じ結果
- **家庭の料理データは Git に入れない**。`seed/` は `.gitignore` 済みで、例外は `seed/sample.csv` のみ。`.env*` も入れない(`.env.example` のみ可)
- UI の文言は日本語。スマホ前提で、幅は `max-w-md`
- 1 Task = 1 Issue = 1 ブランチ = 1 PR。ブランチ名は `task/<N>-<slug>`。PR 本文に `Closes #<Issue番号>`。**main へのマージは人間が行う**。前の Task の PR がマージされてから次の Task に入る
- コミットメッセージの末尾に `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` を付ける。PR 本文の末尾に `🤖 Generated with [Claude Code](https://claude.com/claude-code)` を付ける
- 🧑 印の手順は人間の作業(ブラウザでのアカウント操作など)。実装者はそこで止まり、人間に依頼する

## File Structure

```
src/
  domain/
    menu/
      types.ts          DateStr, DishWithStats, Meal, MenuOptions, MenuDay
      weights.ts        係数の定数
      dates.ts          addDays, daysBetween, isWeekday, nextSunday, todayInTokyo, weekdayLabel
      score.ts          lastEatenBefore, scoreDish
      generate.ts       generateMenu, rerollDay
      options.ts        menuOptionsSchema(Server Action の入力チェック)
    dish-stats.ts       buildDishStats, listMoods(DB の行 → DishWithStats)
    dish-input.ts       dishInputSchema, toDishRecord(フォームと CSV で共通の入力チェック)
    seed-csv.ts         parseSeedCsv
  db/
    schema.ts           Drizzle のテーブル定義
    client.ts           getDb()
    repo.ts             DB アクセス関数(ここ以外から DB に触らない)
  lib/
    auth.ts             AUTH_COOKIE, authToken, expectedToken, isAuthedToken
    require-auth.ts     requireAuth()(Server Action の先頭で呼ぶ)
  proxy.ts              未ログインなら /login へ
  app/
    layout.tsx          下部ナビ、PWA の metadata
    globals.css
    manifest.ts / apple-icon.tsx
    page.tsx / week-list.tsx / actions.ts          今週
    plan/page.tsx / plan-builder.tsx / actions.ts  献立を作る
    dishes/page.tsx / new/page.tsx / [id]/page.tsx / dish-form.tsx / actions.ts  料理
    login/page.tsx / login-form.tsx / actions.ts   ログイン
scripts/seed.ts         CSV の一括投入
seed/sample.csv         サンプル(架空のデータ)
drizzle.config.ts / vitest.config.ts / .env.example
.github/workflows/ci.yml
```

---

### Task 0: GitHub の Milestone と Issue を作る

**Files:** なし(GitHub 上の操作のみ)

- [ ] **Step 1: Milestone を2つ作る**

```bash
gh api repos/pj8898h2/kondate-app/milestones -f title="Slice 1" -f due_on="2026-11-09T23:59:59Z" -f description="マッチングまでのMVP(仮の締切)"
gh api repos/pj8898h2/kondate-app/milestones -f title="Slice 2" -f description="DevDay(1月下旬)まで"
```

- [ ] **Step 2: Slice 1 の Issue を Task 1〜10 の順に作る**(Issue 番号は PR #1 の次から振られるので、Task N = Issue #(N+1) になる)

```bash
for t in \
  "Task 1: Next.js の雛形・Vitest・CI" \
  "Task 2: 日付ユーティリティと点数関数(TDD)" \
  "Task 3: 献立生成と差し替え(TDD)" \
  "Task 4: 料理の統計・入力チェック・CSV 読み込み(TDD)" \
  "Task 5: DB スキーマと接続・リポジトリ層" \
  "Task 6: パスワード認証" \
  "Task 7: 料理画面と初期データの一括投入" \
  "Task 8: 献立を作る画面" \
  "Task 9: 今週の画面(評価・差し替え)" \
  "Task 10: PWA 化と本番公開・受け入れ確認"; do
  gh issue create --repo pj8898h2/kondate-app --title "$t" --milestone "Slice 1" \
    --body "実装計画: docs/superpowers/plans/2026-10-12-kondate-slice1.md の「${t%%:*}」"
done
```

- [ ] **Step 3: Slice 2 の Issue を作る**(本文は1行でよい。中身は Slice 2 の仕様書で決める)

```bash
for t in "買い物リスト" "配偶者との共有" "個人 OCI の VM 上の Postgres へ移行" "外部レシピの URL 取り込み" "LLM による献立の説明と並べ替え" "冷蔵庫の余り物からの提案" "献立から任意の日を除外する"; do
  gh issue create --repo pj8898h2/kondate-app --title "$t" --milestone "Slice 2" --body "spec §3 Slice 2 参照"
done
```

- [ ] **Step 4: 番号を確認する**

Run: `gh issue list --repo pj8898h2/kondate-app --milestone "Slice 1" --json number,title --jq '.[] | "\(.number) \(.title)"'`
Expected: `2 Task 1: ...` から `11 Task 10: ...` まで。ずれていたら、以降の `Closes #` を実際の番号に読み替える

---

### Task 1: Next.js の雛形・Vitest・CI(Issue #2)

**Files:**
- Create: Next.js の雛形一式(`create-next-app` が生成)
- Create: `vitest.config.ts`, `.github/workflows/ci.yml`, `src/domain/smoke.test.ts`
- Modify: `package.json`(scripts), `.gitignore`(生成物と統合), `src/app/globals.css`

**Interfaces:**
- Produces: `npm test` / `npm run lint` / `npm run typecheck` が使える。`@/` が `src/` を指す

- [ ] **Step 1: ブランチを切る**

```bash
cd /c/Users/pj889/kondate-app && git switch main && git pull && git switch -c task/1-scaffold
```

- [ ] **Step 2: 雛形を生成する**

```bash
npx create-next-app@16 . --ts --eslint --tailwind --app --src-dir --import-alias "@/*" --use-npm --yes
```

Expected: `src/app/page.tsx` などができる。既存ファイルとの衝突で止まった場合は、`docs/`・`LICENSE`・`.gitignore` を一時的に `../kondate-tmp/` に移してから再実行し、戻す

- [ ] **Step 3: `.gitignore` を確認・統合する**

Run: `git diff .gitignore`
生成側に上書きされていたら、生成された内容の末尾に次を追記する(すでにあれば何もしない):

```gitignore
# env (secrets) — keep only the example
.env*
!.env.example

# household data — never commit; only the sample is public
seed/*
!seed/sample.csv
```

- [ ] **Step 4: Vitest を入れる**

```bash
npm install -D vitest
```

`vitest.config.ts`:

```ts
import { fileURLToPath } from 'node:url';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  resolve: { alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) } },
  test: { environment: 'node', include: ['src/**/*.test.ts'] },
});
```

- [ ] **Step 5: scripts を追加する**(`package.json` の `scripts` に追記。`lint` は生成されたものをそのまま使う)

```json
"test": "vitest run",
"typecheck": "next typegen && tsc --noEmit"
```

- [ ] **Step 6: 動作確認用のテストを書く**

`src/domain/smoke.test.ts`:

```ts
import { describe, expect, it } from 'vitest';

describe('smoke', () => {
  it('runs vitest', () => {
    expect(1 + 1).toBe(2);
  });
});
```

Run: `npm test && npm run lint && npm run typecheck`
Expected: 3つとも成功。`typecheck` で `next typegen` が見つからない場合は、`"typecheck": "tsc --noEmit"` に変更して再確認する

- [ ] **Step 7: globals.css を最小にする**(雛形のダークモード用の変数を消し、背景色の不一致を防ぐ)

`src/app/globals.css`:

```css
@import "tailwindcss";
```

- [ ] **Step 8: CI を書く**

`.github/workflows/ci.yml`:

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test
```

- [ ] **Step 9: コミットして PR を出す**

```bash
git add -A
git commit -m "chore: scaffold Next.js 16 with Vitest and CI" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push -u origin task/1-scaffold
gh pr create --fill --body "Closes #2

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Expected: PR の CI(check)が緑になる

---

### Task 2: 日付ユーティリティと点数関数(Issue #3)

**Files:**
- Create: `src/domain/menu/types.ts`, `src/domain/menu/weights.ts`, `src/domain/menu/dates.ts`, `src/domain/menu/score.ts`
- Test: `src/domain/menu/dates.test.ts`, `src/domain/menu/score.test.ts`
- Delete: `src/domain/smoke.test.ts`
- Modify: `docs/superpowers/specs/2026-10-12-kondate-mvp-design.md`(Step 9)

**Interfaces:**
- Produces:
  - `type DateStr = string`
  - `type DishWithStats = { id: number; name: string; effort: number; mainIngredient: string | null; taste: number }`(`taste` は評価の平均。評価がなければ初期評価、それもなければ 3)
  - `type Meal = { date: DateStr; dishId: number; taste: number | null }`
  - `type MenuOptions = { startDate: DateStr; days: number; moods: string[] }`
  - `type MenuDay = { date: DateStr; dishId: number; score: number }`
  - `addDays(d: DateStr, n: number): DateStr` / `daysBetween(from: DateStr, to: DateStr): number` / `isWeekday(d: DateStr): boolean` / `nextSunday(today: DateStr): DateStr` / `todayInTokyo(now?: Date): DateStr` / `weekdayLabel(d: DateStr): string`
  - `lastEatenBefore(dishId: number, meals: Meal[], date: DateStr): DateStr | null`
  - `scoreDish(dish: DishWithStats, date: DateStr, history: Meal[], moods: string[]): number`

- [ ] **Step 1: ブランチを切る**

```bash
git switch main && git pull && git switch -c task/2-score
```

- [ ] **Step 2: 型と係数を書く**

`src/domain/menu/types.ts`:

```ts
/** 'YYYY-MM-DD' */
export type DateStr = string;

export type DishWithStats = {
  id: number;
  name: string;
  effort: number;
  mainIngredient: string | null;
  /** 評価の平均。評価がなければ初期評価、それもなければ 3 */
  taste: number;
};

export type Meal = { date: DateStr; dishId: number; taste: number | null };

export type MenuOptions = { startDate: DateStr; days: number; moods: string[] };

export type MenuDay = { date: DateStr; dishId: number; score: number };
```

`src/domain/menu/weights.ts`:

```ts
/** 初期値。Slice 1 で使ってから見直す(spec §6) */
export const WEIGHTS = {
  defaultTaste: 3,
  weekdayHeavyEffort: 4,
  weekdayPenalty: 2,
  recencyWindowDays: 14,
  recencyMaxPenalty: 3,
  moodBonus: 1.5,
} as const;
```

- [ ] **Step 3: 日付のテストを書く**

`src/domain/menu/dates.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { addDays, daysBetween, isWeekday, nextSunday, todayInTokyo, weekdayLabel } from './dates';

describe('dates', () => {
  it('addDays は月をまたげる', () => {
    expect(addDays('2026-10-31', 1)).toBe('2026-11-01');
    expect(addDays('2026-10-18', -1)).toBe('2026-10-17');
  });

  it('daysBetween は日数の差を返す', () => {
    expect(daysBetween('2026-10-01', '2026-10-15')).toBe(14);
  });

  it('isWeekday は月〜金だけ true(2026-10-12 は月曜)', () => {
    expect(isWeekday('2026-10-12')).toBe(true);
    expect(isWeekday('2026-10-16')).toBe(true);
    expect(isWeekday('2026-10-17')).toBe(false);
    expect(isWeekday('2026-10-18')).toBe(false);
  });

  it('nextSunday は次の日曜日を返し、今日が日曜なら今日を返す', () => {
    expect(nextSunday('2026-10-12')).toBe('2026-10-18');
    expect(nextSunday('2026-10-17')).toBe('2026-10-18');
    expect(nextSunday('2026-10-18')).toBe('2026-10-18');
  });

  it('todayInTokyo は JST の日付を返す', () => {
    expect(todayInTokyo(new Date('2026-10-12T14:59:00Z'))).toBe('2026-10-12');
    expect(todayInTokyo(new Date('2026-10-12T15:00:00Z'))).toBe('2026-10-13');
  });

  it('weekdayLabel は曜日の1文字を返す', () => {
    expect(weekdayLabel('2026-10-18')).toBe('日');
    expect(weekdayLabel('2026-10-12')).toBe('月');
  });
});
```

Run: `npx vitest run src/domain/menu/dates.test.ts`
Expected: FAIL(`./dates` が見つからない)

- [ ] **Step 4: 日付を実装する**

`src/domain/menu/dates.ts`:

```ts
import type { DateStr } from './types';

const DAY_MS = 86_400_000;
const toUtcMs = (d: DateStr) => Date.parse(`${d}T00:00:00Z`);
const utcDay = (d: DateStr) => new Date(toUtcMs(d)).getUTCDay();

export function addDays(d: DateStr, n: number): DateStr {
  return new Date(toUtcMs(d) + n * DAY_MS).toISOString().slice(0, 10);
}

export function daysBetween(from: DateStr, to: DateStr): number {
  return Math.round((toUtcMs(to) - toUtcMs(from)) / DAY_MS);
}

export function isWeekday(d: DateStr): boolean {
  const w = utcDay(d);
  return w >= 1 && w <= 5;
}

/** 次の日曜日。今日が日曜なら今日 */
export function nextSunday(today: DateStr): DateStr {
  const w = utcDay(today);
  return addDays(today, w === 0 ? 0 : 7 - w);
}

export function todayInTokyo(now: Date = new Date()): DateStr {
  return new Intl.DateTimeFormat('en-CA', {
    timeZone: 'Asia/Tokyo',
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
  }).format(now);
}

export function weekdayLabel(d: DateStr): string {
  return '日月火水木金土'[utcDay(d)];
}
```

Run: `npx vitest run src/domain/menu/dates.test.ts`
Expected: PASS

- [ ] **Step 5: 点数のテストを書く**

`src/domain/menu/score.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { lastEatenBefore, scoreDish } from './score';
import type { DishWithStats, Meal } from './types';

const dish = (o: Partial<DishWithStats> = {}): DishWithStats => ({
  id: 1, name: 'x', effort: 2, mainIngredient: null, taste: 3, ...o,
});
const SUN = '2026-10-18';
const MON = '2026-10-12';

describe('lastEatenBefore', () => {
  it('指定日より前で最も新しい日を返す', () => {
    const meals: Meal[] = [
      { date: '2026-10-10', dishId: 1, taste: null },
      { date: '2026-10-17', dishId: 1, taste: null },
      { date: '2026-10-18', dishId: 1, taste: null },
      { date: '2026-10-16', dishId: 2, taste: null },
    ];
    expect(lastEatenBefore(1, meals, SUN)).toBe('2026-10-17');
    expect(lastEatenBefore(3, meals, SUN)).toBeNull();
  });
});

describe('scoreDish', () => {
  it('減点も加点もなければ美味しさがそのまま点数になる', () => {
    expect(scoreDish(dish({ taste: 4 }), SUN, [], [])).toBe(4);
  });

  it('平日に手間 4 以上なら -2、休日は減点しない', () => {
    expect(scoreDish(dish({ effort: 4 }), MON, [], [])).toBe(1);
    expect(scoreDish(dish({ effort: 5 }), SUN, [], [])).toBe(3);
    expect(scoreDish(dish({ effort: 3 }), MON, [], [])).toBe(3);
  });

  it('14日以内に食べた料理は日が近いほど大きく減点する', () => {
    const at = (date: string): Meal[] => [{ date, dishId: 1, taste: null }];
    expect(scoreDish(dish(), SUN, at('2026-10-17'), [])).toBeCloseTo(0);
    expect(scoreDish(dish(), SUN, at('2026-10-04'), [])).toBeCloseTo(3 - 3 / 14);
    expect(scoreDish(dish(), SUN, at('2026-10-03'), [])).toBe(3);
  });

  it('当日以降の記録は「最近食べた」に数えない', () => {
    expect(scoreDish(dish(), SUN, [{ date: SUN, dishId: 1, taste: null }], [])).toBe(3);
  });

  it('主食材が気分に合えば +1.5', () => {
    expect(scoreDish(dish({ mainIngredient: '魚' }), SUN, [], ['魚'])).toBe(4.5);
    expect(scoreDish(dish({ mainIngredient: '魚' }), SUN, [], ['ミンチ'])).toBe(3);
    expect(scoreDish(dish({ mainIngredient: null }), SUN, [], ['魚'])).toBe(3);
  });
});
```

Run: `npx vitest run src/domain/menu/score.test.ts`
Expected: FAIL(`./score` が見つからない)

- [ ] **Step 6: 点数を実装する**

`src/domain/menu/score.ts`:

```ts
import { daysBetween, isWeekday } from './dates';
import type { DateStr, DishWithStats, Meal } from './types';
import { WEIGHTS as W } from './weights';

export function lastEatenBefore(dishId: number, meals: Meal[], date: DateStr): DateStr | null {
  let last: DateStr | null = null;
  for (const m of meals) {
    if (m.dishId === dishId && m.date < date && (last === null || m.date > last)) last = m.date;
  }
  return last;
}

export function scoreDish(dish: DishWithStats, date: DateStr, history: Meal[], moods: string[]): number {
  let score = dish.taste;
  if (isWeekday(date) && dish.effort >= W.weekdayHeavyEffort) score -= W.weekdayPenalty;
  const last = lastEatenBefore(dish.id, history, date);
  if (last !== null) {
    const n = daysBetween(last, date);
    if (n <= W.recencyWindowDays) {
      score -= (W.recencyMaxPenalty * (W.recencyWindowDays + 1 - n)) / W.recencyWindowDays;
    }
  }
  if (dish.mainIngredient !== null && moods.includes(dish.mainIngredient)) score += W.moodBonus;
  return score;
}
```

Run: `npx vitest run src/domain/menu/`
Expected: PASS

- [ ] **Step 7: smoke テストを消す**

```bash
git rm src/domain/smoke.test.ts
```

- [ ] **Step 8: 全体を確認する**

Run: `npm test && npm run lint && npm run typecheck`
Expected: 成功

- [ ] **Step 9: 仕様書に、実装計画で決めた3点を反映する**

`docs/superpowers/specs/2026-10-12-kondate-mvp-design.md` を次のように直す:
- §5 `dishes` に `baseTaste: int NULL CHECK (1..5)   // 初期評価。一括投入や登録時に付ける` を追加する。「料理の美味しさ」の説明を「`meals.taste` の平均。評価がなければ `baseTaste`、それもなければ **3**」に変える
- §6「選び方」の `rerollDay` の説明に「画面側で、その日に一度出して断った料理(`skip`)も除外する(同じ2品を行き来しないように)」を追加する
- §7「献立を作る」の開始日の説明を「次の日曜日(今日が日曜なら今日)」に変える

- [ ] **Step 10: コミットして PR を出す**

```bash
git add -A
git commit -m "feat(domain): add date utils and dish scoring" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push -u origin task/2-score
gh pr create --fill --body "Closes #3

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

### Task 3: 献立生成と差し替え(Issue #4)

**Files:**
- Create: `src/domain/menu/generate.ts`, `src/domain/menu/options.ts`
- Test: `src/domain/menu/generate.test.ts`, `src/domain/menu/options.test.ts`

**Interfaces:**
- Consumes: Task 2 の型、`addDays`、`scoreDish`
- Produces:
  - `generateMenu(dishes: DishWithStats[], meals: Meal[], options: MenuOptions): MenuDay[]`
  - `rerollDay(menu: MenuDay[], dayIndex: number, dishes: DishWithStats[], meals: Meal[], options: MenuOptions, skip?: number[]): MenuDay[]`
  - `menuOptionsSchema`(zod。`MenuOptions` を検証する)

- [ ] **Step 1: ブランチを切り、zod を入れる**

```bash
git switch main && git pull && git switch -c task/3-generate
npm install zod
```

- [ ] **Step 2: 生成のテストを書く**

`src/domain/menu/generate.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { generateMenu, rerollDay } from './generate';
import type { DishWithStats, Meal, MenuOptions } from './types';

const dish = (id: number, o: Partial<DishWithStats> = {}): DishWithStats => ({
  id, name: `d${id}`, effort: 2, mainIngredient: null, taste: 3, ...o,
});
const opts = (o: Partial<MenuOptions> = {}): MenuOptions => ({
  startDate: '2026-10-18', days: 1, moods: [], ...o,
});
const ids = (menu: { dishId: number }[]) => menu.map((m) => m.dishId);

describe('generateMenu', () => {
  it('毎日、点数の高い順に重複なしで選び、日付を1日ずつ進める', () => {
    const dishes = [dish(1, { taste: 5 }), dish(2, { taste: 4 }), dish(3, { taste: 3 })];
    const menu = generateMenu(dishes, [], opts({ days: 3 }));
    expect(ids(menu)).toEqual([1, 2, 3]);
    expect(menu.map((m) => m.date)).toEqual(['2026-10-18', '2026-10-19', '2026-10-20']);
  });

  it('平日は手間の重い料理を避け、休日は避けない', () => {
    const dishes = [dish(1, { taste: 5, effort: 5 }), dish(2, { taste: 4, effort: 1 })];
    expect(ids(generateMenu(dishes, [], opts({ startDate: '2026-10-12' })))).toEqual([2]);
    expect(ids(generateMenu(dishes, [], opts({ startDate: '2026-10-18' })))).toEqual([1]);
  });

  it('同点のときは dishId の小さい方を選ぶ', () => {
    expect(ids(generateMenu([dish(5), dish(2)], [], opts()))).toEqual([2]);
  });

  it('料理が日数より少なければ、作れた日数分だけ返す', () => {
    expect(generateMenu([dish(1), dish(2)], [], opts({ days: 3 }))).toHaveLength(2);
  });

  it('気分に合う料理を優先するが、合う料理がなくても献立は組む', () => {
    const fish = dish(2, { taste: 3, mainIngredient: '魚' });
    const meat = dish(1, { taste: 4, mainIngredient: '豚バラ' });
    expect(ids(generateMenu([meat, fish], [], opts({ moods: ['魚'] })))).toEqual([2]);
    expect(ids(generateMenu([meat], [], opts({ moods: ['魚'] })))).toEqual([1]);
  });

  it('最近食べた料理を避ける', () => {
    const meals: Meal[] = [{ date: '2026-10-17', dishId: 1, taste: 5 }];
    const dishes = [dish(1, { taste: 5 }), dish(2, { taste: 4 })];
    expect(ids(generateMenu(dishes, meals, opts()))).toEqual([2]);
  });

  it('開始日以降の記録(作り直す前の確定済み献立)は「最近食べた」に数えない', () => {
    const meals: Meal[] = [{ date: '2026-10-18', dishId: 1, taste: null }];
    const dishes = [dish(1, { taste: 5 }), dish(2, { taste: 4 })];
    expect(ids(generateMenu(dishes, meals, opts()))).toEqual([1]);
  });

  it('同じ入力なら同じ結果になる', () => {
    const dishes = [dish(3), dish(1, { taste: 4 }), dish(2)];
    const o = opts({ days: 3, moods: ['魚'] });
    expect(generateMenu(dishes, [], o)).toEqual(generateMenu(dishes, [], o));
  });
});

describe('rerollDay', () => {
  const dishes = [dish(1, { taste: 5 }), dish(2, { taste: 4 }), dish(3, { taste: 3 }), dish(4, { taste: 2 })];
  const o = opts({ days: 2 });
  const menu = generateMenu(dishes, [], o); // [1, 2]

  it('その日を、献立の他の日と重複しない次点の料理に入れ替える', () => {
    expect(ids(rerollDay(menu, 0, dishes, [], o))).toEqual([3, 2]);
  });

  it('skip に入っている料理も除外する', () => {
    expect(ids(rerollDay(menu, 0, dishes, [], o, [3]))).toEqual([4, 2]);
  });

  it('候補がなければ元のまま返す', () => {
    const two = [dish(1), dish(2)];
    const m = generateMenu(two, [], o);
    expect(rerollDay(m, 0, two, [], o)).toEqual(m);
  });

  it('範囲外の dayIndex なら元のまま返す', () => {
    expect(rerollDay(menu, 5, dishes, [], o)).toEqual(menu);
  });
});
```

Run: `npx vitest run src/domain/menu/generate.test.ts`
Expected: FAIL(`./generate` が見つからない)

- [ ] **Step 3: 生成を実装する**

`src/domain/menu/generate.ts`:

```ts
import { addDays } from './dates';
import { scoreDish } from './score';
import type { DateStr, DishWithStats, Meal, MenuDay, MenuOptions } from './types';

function pickBest(
  dishes: DishWithStats[],
  date: DateStr,
  history: Meal[],
  moods: string[],
  exclude: Set<number>,
): MenuDay | null {
  let best: MenuDay | null = null;
  for (const d of [...dishes].sort((a, b) => a.id - b.id)) {
    if (exclude.has(d.id)) continue;
    const score = scoreDish(d, date, history, moods);
    if (best === null || score > best.score) best = { date, dishId: d.id, score };
  }
  return best;
}

/** 開始日以降の記録は作り直す前の献立なので、「最近食べた」の判定から外す */
const historyBefore = (meals: Meal[], startDate: DateStr) => meals.filter((m) => m.date < startDate);

export function generateMenu(dishes: DishWithStats[], meals: Meal[], options: MenuOptions): MenuDay[] {
  const history = historyBefore(meals, options.startDate);
  const used = new Set<number>();
  const menu: MenuDay[] = [];
  for (let i = 0; i < options.days; i++) {
    const best = pickBest(dishes, addDays(options.startDate, i), history, options.moods, used);
    if (best === null) break;
    used.add(best.dishId);
    menu.push(best);
  }
  return menu;
}

export function rerollDay(
  menu: MenuDay[],
  dayIndex: number,
  dishes: DishWithStats[],
  meals: Meal[],
  options: MenuOptions,
  skip: number[] = [],
): MenuDay[] {
  const target = menu[dayIndex];
  if (target === undefined) return menu;
  const exclude = new Set([...menu.map((m) => m.dishId), ...skip]);
  const best = pickBest(dishes, target.date, historyBefore(meals, options.startDate), options.moods, exclude);
  if (best === null) return menu;
  return menu.map((m, i) => (i === dayIndex ? best : m));
}
```

Run: `npx vitest run src/domain/menu/generate.test.ts`
Expected: PASS

- [ ] **Step 4: 入力チェックのテストを書く**

`src/domain/menu/options.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { menuOptionsSchema } from './options';

describe('menuOptionsSchema', () => {
  it('正しい入力を通す', () => {
    const o = { startDate: '2026-10-18', days: 6, moods: ['魚'] };
    expect(menuOptionsSchema.parse(o)).toEqual(o);
  });

  it('日数は 1〜7、日付は YYYY-MM-DD', () => {
    expect(menuOptionsSchema.safeParse({ startDate: '2026-10-18', days: 8, moods: [] }).success).toBe(false);
    expect(menuOptionsSchema.safeParse({ startDate: '2026-10-18', days: 0, moods: [] }).success).toBe(false);
    expect(menuOptionsSchema.safeParse({ startDate: '10/18', days: 6, moods: [] }).success).toBe(false);
  });
});
```

Run: `npx vitest run src/domain/menu/options.test.ts`
Expected: FAIL

- [ ] **Step 5: 入力チェックを実装する**

`src/domain/menu/options.ts`:

```ts
import { z } from 'zod';

export const menuOptionsSchema = z.object({
  startDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  days: z.number().int().min(1).max(7),
  moods: z.array(z.string()).max(30),
});
```

Run: `npm test && npm run lint && npm run typecheck`
Expected: 成功

- [ ] **Step 6: コミットして PR を出す**

```bash
git add -A
git commit -m "feat(domain): add generateMenu and rerollDay" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push -u origin task/3-generate
gh pr create --fill --body "Closes #4

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

### Task 4: 料理の統計・入力チェック・CSV 読み込み(Issue #5)

**Files:**
- Create: `src/domain/dish-stats.ts`, `src/domain/dish-input.ts`, `src/domain/seed-csv.ts`, `seed/sample.csv`
- Test: `src/domain/dish-stats.test.ts`, `src/domain/dish-input.test.ts`, `src/domain/seed-csv.test.ts`

**Interfaces:**
- Consumes: Task 2 の型、`WEIGHTS.defaultTaste`
- Produces:
  - `type DishRow = { id: number; name: string; effort: number; baseTaste: number | null }`
  - `type IngredientRow = { dishId: number; name: string; isMain: boolean }`
  - `buildDishStats(dishes: DishRow[], ingredients: IngredientRow[], meals: Meal[]): DishWithStats[]`
  - `listMoods(dishes: DishWithStats[]): string[]`
  - `type DishInput = { name: string; effort: number; baseTaste: number | null; mainIngredient: string | null; ingredients: string[]; sourceUrl: string | null; memo: string | null }`
  - `dishInputSchema`(入力はすべて文字列 → 出力は `DishInput`)
  - `toDishRecord(src: Record<string, unknown>): Record<DishField, string>`(足りないキーを `''` で埋める)
  - `parseSeedCsv(text: string): { dishes: DishInput[]; errors: string[] }`

- [ ] **Step 1: ブランチを切り、csv-parse を入れる**

```bash
git switch main && git pull && git switch -c task/4-dish-domain
npm install csv-parse
```

- [ ] **Step 2: 統計のテストを書く**

`src/domain/dish-stats.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { buildDishStats, listMoods } from './dish-stats';

const row = (id: number, baseTaste: number | null = null) => ({ id, name: `d${id}`, effort: 2, baseTaste });

describe('buildDishStats', () => {
  it('美味しさは評価の平均(未評価の記録は無視)', () => {
    const meals = [
      { date: '2026-10-01', dishId: 1, taste: 4 },
      { date: '2026-10-08', dishId: 1, taste: 5 },
      { date: '2026-10-15', dishId: 1, taste: null },
    ];
    expect(buildDishStats([row(1, 2)], [], meals)[0].taste).toBe(4.5);
  });

  it('評価がなければ初期評価、それもなければ 3', () => {
    const [a, b] = buildDishStats([row(1, 4), row(2)], [], []);
    expect(a.taste).toBe(4);
    expect(b.taste).toBe(3);
  });

  it('主食材は isMain の材料。なければ null', () => {
    const ing = [
      { dishId: 1, name: '大根', isMain: false },
      { dishId: 1, name: '豚バラ', isMain: true },
    ];
    const [a, b] = buildDishStats([row(1), row(2)], ing, []);
    expect(a.mainIngredient).toBe('豚バラ');
    expect(b.mainIngredient).toBeNull();
  });
});

describe('listMoods', () => {
  it('主食材を重複なしで並べ、null は除く', () => {
    const dishes = buildDishStats(
      [row(1), row(2), row(3), row(4)],
      [
        { dishId: 1, name: '魚', isMain: true },
        { dishId: 2, name: '豚バラ', isMain: true },
        { dishId: 3, name: '魚', isMain: true },
      ],
      [],
    );
    expect(listMoods(dishes)).toEqual(['豚バラ', '魚']);
  });
});
```

Run: `npx vitest run src/domain/dish-stats.test.ts`
Expected: FAIL

- [ ] **Step 3: 統計を実装する**

`src/domain/dish-stats.ts`:

```ts
import type { DishWithStats, Meal } from './menu/types';
import { WEIGHTS } from './menu/weights';

export type DishRow = { id: number; name: string; effort: number; baseTaste: number | null };
export type IngredientRow = { dishId: number; name: string; isMain: boolean };

export function buildDishStats(dishes: DishRow[], ingredients: IngredientRow[], meals: Meal[]): DishWithStats[] {
  return dishes.map((d) => {
    const rated = meals.filter((m) => m.dishId === d.id && m.taste !== null).map((m) => m.taste as number);
    const taste = rated.length > 0
      ? rated.reduce((a, b) => a + b, 0) / rated.length
      : (d.baseTaste ?? WEIGHTS.defaultTaste);
    const main = ingredients.find((i) => i.dishId === d.id && i.isMain);
    return { id: d.id, name: d.name, effort: d.effort, mainIngredient: main?.name ?? null, taste };
  });
}

export function listMoods(dishes: DishWithStats[]): string[] {
  const set = new Set<string>();
  for (const d of dishes) if (d.mainIngredient !== null) set.add(d.mainIngredient);
  return [...set].sort();
}
```

Run: `npx vitest run src/domain/dish-stats.test.ts`
Expected: PASS

- [ ] **Step 4: 入力チェックのテストを書く**

`src/domain/dish-input.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { dishInputSchema, toDishRecord } from './dish-input';

const base = {
  name: ' 麻婆豆腐 ', effort: '3', baseTaste: '', mainIngredient: 'ミンチ',
  ingredients: '豆腐、ねぎ,豆板醤', sourceUrl: '', memo: '',
};

describe('dishInputSchema', () => {
  it('文字列の入力を DishInput に変換する', () => {
    expect(dishInputSchema.parse(base)).toEqual({
      name: '麻婆豆腐', effort: 3, baseTaste: null, mainIngredient: 'ミンチ',
      ingredients: ['豆腐', 'ねぎ', '豆板醤'], sourceUrl: null, memo: null,
    });
  });

  it('初期評価と URL があれば変換する', () => {
    const r = dishInputSchema.parse({ ...base, baseTaste: '5', sourceUrl: 'https://example.com/r' });
    expect(r.baseTaste).toBe(5);
    expect(r.sourceUrl).toBe('https://example.com/r');
  });

  it('名前が空、手間が範囲外、URL が不正なら失敗する', () => {
    expect(dishInputSchema.safeParse({ ...base, name: '  ' }).success).toBe(false);
    expect(dishInputSchema.safeParse({ ...base, effort: '6' }).success).toBe(false);
    expect(dishInputSchema.safeParse({ ...base, effort: '' }).success).toBe(false);
    expect(dishInputSchema.safeParse({ ...base, baseTaste: '0' }).success).toBe(false);
    expect(dishInputSchema.safeParse({ ...base, sourceUrl: 'not a url' }).success).toBe(false);
  });
});

describe('toDishRecord', () => {
  it('足りないキーを空文字で埋め、文字列以外も空文字にする', () => {
    const r = toDishRecord({ name: 'x', effort: '2', memo: 123 });
    expect(r).toEqual({ name: 'x', effort: '2', baseTaste: '', mainIngredient: '', ingredients: '', sourceUrl: '', memo: '' });
    expect(dishInputSchema.safeParse(r).success).toBe(true);
  });
});
```

Run: `npx vitest run src/domain/dish-input.test.ts`
Expected: FAIL

- [ ] **Step 5: 入力チェックを実装する**

`src/domain/dish-input.ts`:

```ts
import { z } from 'zod';

const RATING_MSG = '1〜5 で入力してください';
const optionalText = z.string().trim().transform((s) => (s === '' ? null : s));
const rating = z.coerce.number().int(RATING_MSG).min(1, RATING_MSG).max(5, RATING_MSG);

export const dishInputSchema = z.object({
  name: z.string().trim().min(1, '名前を入力してください'),
  effort: rating,
  baseTaste: z.union([z.literal('').transform(() => null), rating]),
  mainIngredient: optionalText,
  ingredients: z.string().transform((s) => s.split(/[、,;\n]/).map((x) => x.trim()).filter(Boolean)),
  sourceUrl: z.union([z.literal('').transform(() => null), z.url('URL の形式が正しくありません')]),
  memo: optionalText,
});

export type DishInput = z.infer<typeof dishInputSchema>;

const FIELDS = ['name', 'effort', 'baseTaste', 'mainIngredient', 'ingredients', 'sourceUrl', 'memo'] as const;
export type DishField = (typeof FIELDS)[number];

/** フォームや CSV の値を、全キーが文字列のレコードにそろえる */
export function toDishRecord(src: Record<string, unknown>): Record<DishField, string> {
  const out = {} as Record<DishField, string>;
  for (const f of FIELDS) out[f] = typeof src[f] === 'string' ? (src[f] as string) : '';
  return out;
}
```

Run: `npx vitest run src/domain/dish-input.test.ts`
Expected: PASS(`effort: ''` は `z.coerce.number` で 0 になり、`min(1)` で失敗する)

- [ ] **Step 6: サンプル CSV を作る**(架空のデータ。実データは `seed/dishes.csv` に置き、Git には入らない)

`seed/sample.csv`:

```csv
name,effort,base_taste,main,ingredients,source_url,memo
豚バラ大根,2,4,豚バラ,大根、しょうゆ、みりん,,
鮭のホイル焼き,2,4,鮭,玉ねぎ、きのこ、バター,,
麻婆豆腐,3,5,ミンチ,豆腐、ねぎ、豆板醤,,
ささみのチーズ焼き,1,3,ささみ,チーズ、大葉,,
```

- [ ] **Step 7: CSV 読み込みのテストを書く**

`src/domain/seed-csv.test.ts`:

```ts
import { readFileSync } from 'node:fs';
import { describe, expect, it } from 'vitest';
import { parseSeedCsv } from './seed-csv';

describe('parseSeedCsv', () => {
  it('サンプル CSV を DishInput の配列にする', () => {
    const { dishes, errors } = parseSeedCsv(readFileSync('seed/sample.csv', 'utf8'));
    expect(errors).toEqual([]);
    expect(dishes).toHaveLength(4);
    expect(dishes[0]).toEqual({
      name: '豚バラ大根', effort: 2, baseTaste: 4, mainIngredient: '豚バラ',
      ingredients: ['大根', 'しょうゆ', 'みりん'], sourceUrl: null, memo: null,
    });
  });

  it('不正な行は行番号付きでエラーにする', () => {
    const text = 'name,effort,base_taste,main,ingredients,source_url,memo\nOK,2,,,,,\nNG,9,,,,,\n';
    const { dishes, errors } = parseSeedCsv(text);
    expect(dishes).toHaveLength(1);
    expect(errors).toHaveLength(1);
    expect(errors[0]).toContain('3行目');
  });
});
```

Run: `npx vitest run src/domain/seed-csv.test.ts`
Expected: FAIL

- [ ] **Step 8: CSV 読み込みを実装する**

`src/domain/seed-csv.ts`:

```ts
import { parse } from 'csv-parse/sync';
import { dishInputSchema, toDishRecord, type DishInput } from './dish-input';

const COLUMN_TO_FIELD: Record<string, string> = {
  name: 'name', effort: 'effort', base_taste: 'baseTaste', main: 'mainIngredient',
  ingredients: 'ingredients', source_url: 'sourceUrl', memo: 'memo',
};

export function parseSeedCsv(text: string): { dishes: DishInput[]; errors: string[] } {
  const rows: Record<string, string>[] = parse(text, { columns: true, skip_empty_lines: true, trim: true, bom: true });
  const dishes: DishInput[] = [];
  const errors: string[] = [];
  rows.forEach((row, i) => {
    const mapped: Record<string, unknown> = {};
    for (const [col, value] of Object.entries(row)) {
      const field = COLUMN_TO_FIELD[col];
      if (field) mapped[field] = value;
    }
    const result = dishInputSchema.safeParse(toDishRecord(mapped));
    if (result.success) dishes.push(result.data);
    else errors.push(`${i + 2}行目: ${result.error.issues.map((x) => x.message).join(' / ')}`);
  });
  return { dishes, errors };
}
```

Run: `npm test && npm run lint && npm run typecheck`
Expected: 成功

- [ ] **Step 9: コミットして PR を出す**

```bash
git add -A
git commit -m "feat(domain): add dish stats, input schema and seed CSV parser" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push -u origin task/4-dish-domain
gh pr create --fill --body "Closes #5

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

### Task 5: DB スキーマと接続・リポジトリ層(Issue #6)

**Files:**
- Create: `src/db/schema.ts`, `src/db/client.ts`, `src/db/repo.ts`, `drizzle.config.ts`, `.env.example`, `drizzle/`(生成物)
- Modify: `package.json`(scripts)

**Interfaces:**
- Consumes: `buildDishStats`, `DishInput`, Task 2 の型
- Produces(すべて async):
  - `getDishStats(): Promise<{ dishes: DishWithStats[]; meals: Meal[] }>`
  - `getDishForEdit(id: number): Promise<{ id: number; name: string; effort: number; baseTaste: number | null; sourceUrl: string | null; memo: string | null; mainIngredient: string; ingredients: string[] } | null>`
  - `createDish(input: DishInput): Promise<number>` / `updateDish(id: number, input: DishInput): Promise<void>`
  - `confirmMenu(days: { date: DateStr; dishId: number }[]): Promise<void>`
  - `setTaste(date: DateStr, taste: number): Promise<void>` / `replaceMealDish(date: DateStr, dishId: number): Promise<void>`
  - `listMealsBetween(from: DateStr, to: DateStr): Promise<{ date: DateStr; dishId: number; taste: number | null; dishName: string }[]>`
  - `listDishNames(): Promise<{ id: number; name: string }[]>`

- [ ] **Step 1: 🧑 Supabase のプロジェクトを作る**
  1. https://supabase.com で新規プロジェクトを作る(リージョンは Tokyo、DB パスワードは控えておく)
  2. 「Connect」→「Transaction pooler」の接続文字列(ポート 6543)をコピーする
  3. `C:\Users\pj889\kondate-app\.env.local` に次を書く(このファイルは Git に入らない):

```
DATABASE_URL=postgresql://postgres.xxxx:<password>@aws-0-ap-northeast-1.pooler.supabase.com:6543/postgres
APP_PASSWORD=<家で使うパスワード>
AUTH_SECRET=<ランダムな長い文字列。例: node -e "console.log(crypto.randomUUID()+crypto.randomUUID())">
```

- [ ] **Step 2: ブランチを切り、依存を入れる**

```bash
git switch main && git pull && git switch -c task/5-db
npm install drizzle-orm postgres
npm install -D drizzle-kit tsx
```

- [ ] **Step 3: `.env.example` を作る**

```
DATABASE_URL=postgresql://USER:PASSWORD@HOST:6543/postgres
APP_PASSWORD=change-me
AUTH_SECRET=change-me-to-a-long-random-string
```

- [ ] **Step 4: スキーマを書く**

`src/db/schema.ts`:

```ts
import { sql } from 'drizzle-orm';
import { boolean, check, date, integer, pgTable, serial, text, timestamp } from 'drizzle-orm/pg-core';

export const dishes = pgTable(
  'dishes',
  {
    id: serial('id').primaryKey(),
    name: text('name').notNull(),
    effort: integer('effort').notNull(),
    baseTaste: integer('base_taste'),
    sourceUrl: text('source_url'),
    memo: text('memo'),
    createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    check('dishes_effort_range', sql`${t.effort} between 1 and 5`),
    check('dishes_base_taste_range', sql`${t.baseTaste} between 1 and 5`),
  ],
);

export const ingredients = pgTable('ingredients', {
  id: serial('id').primaryKey(),
  dishId: integer('dish_id').notNull().references(() => dishes.id, { onDelete: 'cascade' }),
  name: text('name').notNull(),
  isMain: boolean('is_main').notNull().default(false),
});

export const meals = pgTable(
  'meals',
  {
    id: serial('id').primaryKey(),
    date: date('date', { mode: 'string' }).notNull().unique(),
    dishId: integer('dish_id').notNull().references(() => dishes.id),
    taste: integer('taste'),
  },
  (t) => [check('meals_taste_range', sql`${t.taste} between 1 and 5`)],
);
```

- [ ] **Step 5: 接続を書く**(import しただけでは接続しない。CI のテストで DB を触らないため)

`src/db/client.ts`:

```ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema';

function create() {
  const url = process.env.DATABASE_URL;
  if (!url) throw new Error('DATABASE_URL is not set');
  // Supabase の transaction pooler では prepared statement が使えない
  return drizzle(postgres(url, { prepare: false, max: 1 }), { schema });
}

let db: ReturnType<typeof create> | undefined;

export function getDb() {
  db ??= create();
  return db;
}
```

- [ ] **Step 6: drizzle-kit の設定と scripts**

`drizzle.config.ts`:

```ts
import { defineConfig } from 'drizzle-kit';

try {
  process.loadEnvFile('.env.local');
} catch {
  // .env.local がない環境(CI など)では環境変数をそのまま使う
}

export default defineConfig({
  schema: './src/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: { url: process.env.DATABASE_URL ?? '' },
});
```

`package.json` の `scripts` に追記:

```json
"db:generate": "drizzle-kit generate",
"db:migrate": "drizzle-kit migrate",
"seed": "tsx --env-file=.env.local scripts/seed.ts"
```

- [ ] **Step 7: マイグレーションを作って適用する**

```bash
npm run db:generate
npm run db:migrate
```

Expected: `drizzle/0000_*.sql` が生成され、Supabase の Table Editor に `dishes` `ingredients` `meals` が見える。transaction pooler で migrate が失敗したら、Supabase の「Session pooler」(ポート 5432)の URL を一時的に `DATABASE_URL` に入れて再実行する

- [ ] **Step 8: リポジトリ層を書く**

`src/db/repo.ts`:

```ts
import { asc, between, eq, sql } from 'drizzle-orm';
import type { DishInput } from '@/domain/dish-input';
import { buildDishStats } from '@/domain/dish-stats';
import type { DateStr, DishWithStats, Meal } from '@/domain/menu/types';
import { getDb } from './client';
import { dishes, ingredients, meals } from './schema';

export async function getDishStats(): Promise<{ dishes: DishWithStats[]; meals: Meal[] }> {
  const db = getDb();
  const [dishRows, ingredientRows, mealRows] = await Promise.all([
    db.select({ id: dishes.id, name: dishes.name, effort: dishes.effort, baseTaste: dishes.baseTaste })
      .from(dishes).orderBy(asc(dishes.id)),
    db.select({ dishId: ingredients.dishId, name: ingredients.name, isMain: ingredients.isMain }).from(ingredients),
    db.select({ date: meals.date, dishId: meals.dishId, taste: meals.taste }).from(meals),
  ]);
  return { dishes: buildDishStats(dishRows, ingredientRows, mealRows), meals: mealRows };
}

export async function getDishForEdit(id: number) {
  const db = getDb();
  const [dish] = await db.select().from(dishes).where(eq(dishes.id, id));
  if (!dish) return null;
  const rows = await db.select().from(ingredients).where(eq(ingredients.dishId, id));
  return {
    id: dish.id, name: dish.name, effort: dish.effort, baseTaste: dish.baseTaste,
    sourceUrl: dish.sourceUrl, memo: dish.memo,
    mainIngredient: rows.find((r) => r.isMain)?.name ?? '',
    ingredients: rows.filter((r) => !r.isMain).map((r) => r.name),
  };
}

function ingredientRowsFor(dishId: number, input: DishInput) {
  const rows: { dishId: number; name: string; isMain: boolean }[] = [];
  if (input.mainIngredient !== null) rows.push({ dishId, name: input.mainIngredient, isMain: true });
  for (const name of input.ingredients) {
    if (name !== input.mainIngredient) rows.push({ dishId, name, isMain: false });
  }
  return rows;
}

const dishValues = (input: DishInput) => ({
  name: input.name, effort: input.effort, baseTaste: input.baseTaste, sourceUrl: input.sourceUrl, memo: input.memo,
});

export async function createDish(input: DishInput): Promise<number> {
  return getDb().transaction(async (tx) => {
    const [row] = await tx.insert(dishes).values(dishValues(input)).returning({ id: dishes.id });
    const rows = ingredientRowsFor(row.id, input);
    if (rows.length > 0) await tx.insert(ingredients).values(rows);
    return row.id;
  });
}

export async function updateDish(id: number, input: DishInput): Promise<void> {
  await getDb().transaction(async (tx) => {
    await tx.update(dishes).set(dishValues(input)).where(eq(dishes.id, id));
    await tx.delete(ingredients).where(eq(ingredients.dishId, id));
    const rows = ingredientRowsFor(id, input);
    if (rows.length > 0) await tx.insert(ingredients).values(rows);
  });
}

/** 確定。同じ日付の記録があれば料理を置き換え、評価を消す(spec §6) */
export async function confirmMenu(days: { date: DateStr; dishId: number }[]): Promise<void> {
  if (days.length === 0) return;
  await getDb()
    .insert(meals)
    .values(days.map((d) => ({ date: d.date, dishId: d.dishId, taste: null })))
    .onConflictDoUpdate({ target: meals.date, set: { dishId: sql`excluded.dish_id`, taste: sql`null` } });
}

export async function setTaste(date: DateStr, taste: number): Promise<void> {
  await getDb().update(meals).set({ taste }).where(eq(meals.date, date));
}

export async function replaceMealDish(date: DateStr, dishId: number): Promise<void> {
  await getDb().update(meals).set({ dishId, taste: null }).where(eq(meals.date, date));
}

export async function listMealsBetween(from: DateStr, to: DateStr) {
  return getDb()
    .select({ date: meals.date, dishId: meals.dishId, taste: meals.taste, dishName: dishes.name })
    .from(meals)
    .innerJoin(dishes, eq(meals.dishId, dishes.id))
    .where(between(meals.date, from, to))
    .orderBy(asc(meals.date));
}

export async function listDishNames(): Promise<{ id: number; name: string }[]> {
  return getDb().select({ id: dishes.id, name: dishes.name }).from(dishes).orderBy(asc(dishes.name));
}
```

- [ ] **Step 9: 確認する**

Run: `npm test && npm run lint && npm run typecheck`
Expected: 成功(リポジトリ層の動作は Task 7 の画面で手動確認する。Slice 1 では DB を使う結合テストは書かない)

- [ ] **Step 10: コミットして PR を出す**(`.env.local` が `git status` に出ていないことを必ず確認する)

```bash
git status --short   # .env.local が出ていないこと
git add -A
git commit -m "feat(db): add Drizzle schema, migration and repository layer" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push -u origin task/5-db
gh pr create --fill --body "Closes #6

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

### Task 6: パスワード認証(Issue #7)

**Files:**
- Create: `src/lib/auth.ts`, `src/lib/require-auth.ts`, `src/proxy.ts`, `src/app/login/page.tsx`, `src/app/login/login-form.tsx`, `src/app/login/actions.ts`
- Test: `src/lib/auth.test.ts`

**Interfaces:**
- Produces: `AUTH_COOKIE: string` / `authToken(password: string, secret: string): Promise<string>` / `expectedToken(): Promise<string>` / `isAuthedToken(token: string | undefined): Promise<boolean>` / `requireAuth(): Promise<void>`(Server Action の先頭で呼ぶ。未ログインなら throw)

- [ ] **Step 1: ブランチを切る**(`vitest.config.ts` の `include: ['src/**/*.test.ts']` で `src/lib` も対象になるので、設定の変更は不要)

```bash
git switch main && git pull && git switch -c task/6-auth
```

- [ ] **Step 2: テストを書く**

`src/lib/auth.test.ts`:

```ts
import { afterEach, describe, expect, it, vi } from 'vitest';
import { authToken, isAuthedToken } from './auth';

afterEach(() => vi.unstubAllEnvs());

describe('authToken', () => {
  it('同じ入力なら同じ 64 桁の16進数を返す', async () => {
    const a = await authToken('pw', 'secret');
    expect(a).toMatch(/^[0-9a-f]{64}$/);
    expect(await authToken('pw', 'secret')).toBe(a);
  });

  it('パスワードか secret が違えば別の値になる', async () => {
    const a = await authToken('pw', 'secret');
    expect(await authToken('pw2', 'secret')).not.toBe(a);
    expect(await authToken('pw', 'secret2')).not.toBe(a);
  });
});

describe('isAuthedToken', () => {
  it('環境変数から作った値と一致するときだけ true', async () => {
    vi.stubEnv('APP_PASSWORD', 'pw');
    vi.stubEnv('AUTH_SECRET', 'secret');
    expect(await isAuthedToken(await authToken('pw', 'secret'))).toBe(true);
    expect(await isAuthedToken('wrong')).toBe(false);
    expect(await isAuthedToken(undefined)).toBe(false);
  });
});
```

Run: `npx vitest run src/lib/auth.test.ts`
Expected: FAIL

- [ ] **Step 3: 実装する**

`src/lib/auth.ts`:

```ts
export const AUTH_COOKIE = 'kondate_auth';

export async function authToken(password: string, secret: string): Promise<string> {
  const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(`${secret}:${password}`));
  return Array.from(new Uint8Array(digest), (b) => b.toString(16).padStart(2, '0')).join('');
}

export async function expectedToken(): Promise<string> {
  const password = process.env.APP_PASSWORD;
  const secret = process.env.AUTH_SECRET;
  if (!password || !secret) throw new Error('APP_PASSWORD / AUTH_SECRET is not set');
  return authToken(password, secret);
}

export async function isAuthedToken(token: string | undefined): Promise<boolean> {
  return token !== undefined && token === (await expectedToken());
}
```

Run: `npx vitest run src/lib/auth.test.ts`
Expected: PASS

- [ ] **Step 4: Server Action 用のチェックを書く**

`src/lib/require-auth.ts`:

```ts
import { cookies } from 'next/headers';
import { AUTH_COOKIE, isAuthedToken } from './auth';

/** proxy は楽観的なチェックなので、データを変える処理の先頭でも確認する */
export async function requireAuth(): Promise<void> {
  const store = await cookies();
  if (!(await isAuthedToken(store.get(AUTH_COOKIE)?.value))) throw new Error('Unauthorized');
}
```

- [ ] **Step 5: proxy を書く**

`src/proxy.ts`:

```ts
import { NextResponse, type NextRequest } from 'next/server';
import { AUTH_COOKIE, isAuthedToken } from '@/lib/auth';

export async function proxy(request: NextRequest) {
  if (await isAuthedToken(request.cookies.get(AUTH_COOKIE)?.value)) return NextResponse.next();
  return NextResponse.redirect(new URL('/login', request.url));
}

export const config = {
  matcher: ['/((?!login|_next/static|_next/image|favicon.ico|manifest.webmanifest|apple-icon).*)'],
};
```

- [ ] **Step 6: ログイン画面を書く**

`src/app/login/actions.ts`:

```ts
'use server';

import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import { AUTH_COOKIE, expectedToken } from '@/lib/auth';

export type LoginState = { error?: string };

export async function login(_prev: LoginState, formData: FormData): Promise<LoginState> {
  const password = formData.get('password');
  if (typeof password !== 'string' || password !== process.env.APP_PASSWORD) {
    return { error: 'パスワードが違います' };
  }
  (await cookies()).set(AUTH_COOKIE, await expectedToken(), {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    path: '/',
    maxAge: 60 * 60 * 24 * 180,
  });
  redirect('/');
}
```

`src/app/login/login-form.tsx`:

```tsx
'use client';

import { useActionState } from 'react';
import { login, type LoginState } from './actions';

export function LoginForm() {
  const [state, action, pending] = useActionState<LoginState, FormData>(login, {});
  return (
    <form action={action} className="space-y-3">
      <input name="password" type="password" required autoComplete="current-password"
        className="w-full rounded border p-3" placeholder="パスワード" />
      {state.error && <p className="text-sm text-red-600">{state.error}</p>}
      <button disabled={pending} className="w-full rounded bg-orange-500 p-3 text-white disabled:opacity-50">
        ログイン
      </button>
    </form>
  );
}
```

`src/app/login/page.tsx`:

```tsx
import { LoginForm } from './login-form';

export default function LoginPage() {
  return (
    <main className="mx-auto max-w-md p-4">
      <h1 className="mb-4 text-xl font-bold">献立</h1>
      <LoginForm />
    </main>
  );
}
```

- [ ] **Step 7: 手動で確認する**

Run: `npm run dev` → http://localhost:3000 を開く
Expected: `/login` に飛ばされる。違うパスワードなら「パスワードが違います」、正しいパスワードなら `/` に戻る(この時点の `/` は雛形のページ)

- [ ] **Step 8: 確認してコミット・PR**

```bash
npm test && npm run lint && npm run typecheck
git add -A
git commit -m "feat(auth): add single-password login with proxy" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push -u origin task/6-auth
gh pr create --fill --body "Closes #7

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

### Task 7: 料理画面と初期データの一括投入(Issue #8)

**Files:**
- Create: `src/app/dishes/page.tsx`, `src/app/dishes/new/page.tsx`, `src/app/dishes/[id]/page.tsx`, `src/app/dishes/dish-form.tsx`, `src/app/dishes/actions.ts`, `scripts/seed.ts`
- Modify: `src/app/layout.tsx`(下部ナビ)

**Interfaces:**
- Consumes: `getDishStats`, `getDishForEdit`, `createDish`, `updateDish`, `dishInputSchema`, `toDishRecord`, `requireAuth`, `lastEatenBefore`, `addDays`, `todayInTokyo`, `parseSeedCsv`
- Produces: `saveDish(id: number | null, prev: DishFormState, formData: FormData): Promise<DishFormState>`

- [ ] **Step 1: ブランチを切る**

```bash
git switch main && git pull && git switch -c task/7-dishes
```

- [ ] **Step 2: レイアウトに下部ナビを入れる**(雛形の内容を置き換える)

`src/app/layout.tsx`:

```tsx
import type { Metadata, Viewport } from 'next';
import Link from 'next/link';
import './globals.css';

export const metadata: Metadata = {
  title: '献立',
  appleWebApp: { capable: true, title: '献立', statusBarStyle: 'default' },
};

export const viewport: Viewport = { themeColor: '#f97316' };

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body className="bg-white pb-16 text-gray-900">
        {children}
        <nav className="fixed inset-x-0 bottom-0 flex justify-around border-t bg-white py-3 text-sm">
          <Link href="/">今週</Link>
          <Link href="/plan">献立を作る</Link>
          <Link href="/dishes">料理</Link>
        </nav>
      </body>
    </html>
  );
}
```

- [ ] **Step 3: Server Action を書く**

`src/app/dishes/actions.ts`:

```ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { createDish, updateDish } from '@/db/repo';
import { dishInputSchema, toDishRecord } from '@/domain/dish-input';
import { requireAuth } from '@/lib/require-auth';

export type DishFormState = { error?: string };

export async function saveDish(id: number | null, _prev: DishFormState, formData: FormData): Promise<DishFormState> {
  await requireAuth();
  const parsed = dishInputSchema.safeParse(toDishRecord(Object.fromEntries(formData)));
  if (!parsed.success) return { error: parsed.error.issues.map((i) => i.message).join(' / ') };
  if (id === null) await createDish(parsed.data);
  else await updateDish(id, parsed.data);
  revalidatePath('/dishes');
  redirect('/dishes');
}
```

- [ ] **Step 4: フォームを書く**

`src/app/dishes/dish-form.tsx`:

```tsx
'use client';

import { useActionState } from 'react';
import type { DishFormState } from './actions';

type Defaults = {
  name: string; effort: number; baseTaste: number | null; mainIngredient: string;
  ingredients: string[]; sourceUrl: string | null; memo: string | null;
};

const EMPTY: Defaults = {
  name: '', effort: 2, baseTaste: null, mainIngredient: '', ingredients: [], sourceUrl: null, memo: null,
};

export function DishForm({
  action,
  defaults = EMPTY,
}: {
  action: (prev: DishFormState, formData: FormData) => Promise<DishFormState>;
  defaults?: Defaults;
}) {
  const [state, formAction, pending] = useActionState(action, {});
  const field = 'w-full rounded border p-2';
  return (
    <form action={formAction} className="space-y-3">
      <label className="block">名前<input name="name" required defaultValue={defaults.name} className={field} /></label>
      <label className="block">手間(1=らく 〜 5=大変)
        <select name="effort" defaultValue={String(defaults.effort)} className={field}>
          {[1, 2, 3, 4, 5].map((n) => <option key={n} value={n}>{n}</option>)}
        </select>
      </label>
      <label className="block">初期評価(美味しさ。空欄可)
        <select name="baseTaste" defaultValue={defaults.baseTaste === null ? '' : String(defaults.baseTaste)} className={field}>
          <option value="">なし</option>
          {[1, 2, 3, 4, 5].map((n) => <option key={n} value={n}>{n}</option>)}
        </select>
      </label>
      <label className="block">主食材(気分で選ぶときに使う。例: 魚、ミンチ)
        <input name="mainIngredient" defaultValue={defaults.mainIngredient} className={field} />
      </label>
      <label className="block">その他の材料(「、」区切り)
        <input name="ingredients" defaultValue={defaults.ingredients.join('、')} className={field} />
      </label>
      <label className="block">レシピの URL<input name="sourceUrl" type="url" defaultValue={defaults.sourceUrl ?? ''} className={field} /></label>
      <label className="block">メモ<textarea name="memo" defaultValue={defaults.memo ?? ''} className={field} /></label>
      {state.error && <p className="text-sm text-red-600">{state.error}</p>}
      <button disabled={pending} className="w-full rounded bg-orange-500 p-3 text-white disabled:opacity-50">保存</button>
    </form>
  );
}
```

- [ ] **Step 5: 一覧・新規・編集のページを書く**

`src/app/dishes/page.tsx`:

```tsx
import Link from 'next/link';
import { getDishStats } from '@/db/repo';
import { addDays, todayInTokyo } from '@/domain/menu/dates';
import { lastEatenBefore } from '@/domain/menu/score';

export const dynamic = 'force-dynamic';

export default async function DishesPage() {
  const { dishes, meals } = await getDishStats();
  const tomorrow = addDays(todayInTokyo(), 1);
  return (
    <main className="mx-auto max-w-md p-4">
      <div className="mb-4 flex items-center justify-between">
        <h1 className="text-xl font-bold">料理({dishes.length})</h1>
        <Link href="/dishes/new" className="rounded bg-orange-500 px-3 py-2 text-white">+ 追加</Link>
      </div>
      <ul className="divide-y">
        {dishes.map((d) => (
          <li key={d.id}>
            <Link href={`/dishes/${d.id}`} className="flex items-center justify-between py-3">
              <span>
                {d.name}
                {d.mainIngredient && <span className="ml-2 text-xs text-gray-500">{d.mainIngredient}</span>}
              </span>
              <span className="text-xs text-gray-500">
                手間{d.effort} / 味{d.taste.toFixed(1)} / {lastEatenBefore(d.id, meals, tomorrow) ?? '未'}
              </span>
            </Link>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

`src/app/dishes/new/page.tsx`:

```tsx
import { saveDish } from '../actions';
import { DishForm } from '../dish-form';

export default function NewDishPage() {
  return (
    <main className="mx-auto max-w-md p-4">
      <h1 className="mb-4 text-xl font-bold">料理を追加</h1>
      <DishForm action={saveDish.bind(null, null)} />
    </main>
  );
}
```

`src/app/dishes/[id]/page.tsx`:

```tsx
import { notFound } from 'next/navigation';
import { getDishForEdit } from '@/db/repo';
import { saveDish } from '../actions';
import { DishForm } from '../dish-form';

export const dynamic = 'force-dynamic';

export default async function EditDishPage({ params }: { params: Promise<{ id: string }> }) {
  const id = Number((await params).id);
  if (!Number.isInteger(id)) notFound();
  const dish = await getDishForEdit(id);
  if (!dish) notFound();
  return (
    <main className="mx-auto max-w-md p-4">
      <h1 className="mb-4 text-xl font-bold">料理を編集</h1>
      <DishForm action={saveDish.bind(null, id)} defaults={dish} />
    </main>
  );
}
```

- [ ] **Step 6: 一括投入スクリプトを書く**

`scripts/seed.ts`:

```ts
import { readFileSync } from 'node:fs';
import { createDish } from '../src/db/repo';
import { parseSeedCsv } from '../src/domain/seed-csv';

async function main() {
  const path = process.argv[2] ?? 'seed/dishes.csv';
  const { dishes, errors } = parseSeedCsv(readFileSync(path, 'utf8'));
  if (errors.length > 0) {
    console.error(errors.join('\n'));
    process.exit(1);
  }
  for (const d of dishes) await createDish(d);
  console.log(`${dishes.length} 件を登録しました(${path})`);
  process.exit(0);
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

- [ ] **Step 7: サンプルで動作確認する**

Run: `npm run seed -- seed/sample.csv`
Expected: `4 件を登録しました(seed/sample.csv)`。`tsx` が `@/` を解決できずに失敗したら、`src/db/repo.ts` の `@/domain/...` の import を相対パス(`../domain/...`)に変えて再実行する

Run: `npm run dev` → `/dishes`
Expected: 4品が並ぶ。1品を開き、手間を変えて保存すると一覧に反映される。名前を空にして保存すると「名前を入力してください」と出る。新規追加もできる

- [ ] **Step 8: 🧑 実データを投入する**
  1. Supabase の Table Editor で、サンプルの4品を消す(`ingredients` は連鎖で消える)
  2. 手元の料理データを `seed/dishes.csv`(`seed/sample.csv` と同じ列)に整える。Claude に整形を頼んでもよい
  3. `npm run seed` を **1回だけ**実行する(2回実行すると料理が重複する)

- [ ] **Step 9: 確認してコミット・PR**(`git status` に `seed/dishes.csv` が出ていないことを確認する)

```bash
npm test && npm run lint && npm run typecheck
git status --short   # seed/dishes.csv が出ていないこと
git add -A
git commit -m "feat(dishes): add dish list/form and seed script" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push -u origin task/7-dishes
gh pr create --fill --body "Closes #8

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

### Task 8: 献立を作る画面(Issue #9)

**Files:**
- Create: `src/app/plan/page.tsx`, `src/app/plan/plan-builder.tsx`, `src/app/plan/actions.ts`

**Interfaces:**
- Consumes: `getDishStats`, `confirmMenu`, `generateMenu`, `rerollDay`, `menuOptionsSchema`, `listMoods`, `nextSunday`, `todayInTokyo`, `weekdayLabel`, `requireAuth`
- Produces: `type PlanDay = MenuDay & { dishName: string }` / `generatePlan(options: MenuOptions): Promise<PlanDay[]>` / `rerollPlan(menu: MenuDay[], dayIndex: number, options: MenuOptions, skip: number[]): Promise<PlanDay[]>` / `confirmPlan(menu: MenuDay[]): Promise<void>`

- [ ] **Step 1: ブランチを切る**

```bash
git switch main && git pull && git switch -c task/8-plan
```

- [ ] **Step 2: Server Action を書く**

`src/app/plan/actions.ts`:

```ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { z } from 'zod';
import { confirmMenu, getDishStats } from '@/db/repo';
import { generateMenu, rerollDay } from '@/domain/menu/generate';
import { menuOptionsSchema } from '@/domain/menu/options';
import type { DishWithStats, MenuDay, MenuOptions } from '@/domain/menu/types';
import { requireAuth } from '@/lib/require-auth';

export type PlanDay = MenuDay & { dishName: string };

const menuSchema = z.array(
  z.object({ date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/), dishId: z.number().int(), score: z.number() }),
).max(7);

function withNames(menu: MenuDay[], dishes: DishWithStats[]): PlanDay[] {
  const names = new Map(dishes.map((d) => [d.id, d.name]));
  return menu.map((m) => ({ date: m.date, dishId: m.dishId, score: m.score, dishName: names.get(m.dishId) ?? '?' }));
}

export async function generatePlan(options: MenuOptions): Promise<PlanDay[]> {
  await requireAuth();
  const o = menuOptionsSchema.parse(options);
  const { dishes, meals } = await getDishStats();
  return withNames(generateMenu(dishes, meals, o), dishes);
}

export async function rerollPlan(menu: MenuDay[], dayIndex: number, options: MenuOptions, skip: number[]): Promise<PlanDay[]> {
  await requireAuth();
  const o = menuOptionsSchema.parse(options);
  const m = menuSchema.parse(menu);
  const { dishes, meals } = await getDishStats();
  return withNames(rerollDay(m, dayIndex, dishes, meals, o, z.array(z.number().int()).parse(skip)), dishes);
}

export async function confirmPlan(menu: MenuDay[]): Promise<void> {
  await requireAuth();
  const m = menuSchema.parse(menu);
  await confirmMenu(m.map(({ date, dishId }) => ({ date, dishId })));
  revalidatePath('/');
  redirect('/');
}
```

- [ ] **Step 3: 画面の部品を書く**

`src/app/plan/plan-builder.tsx`:

```tsx
'use client';

import { useState, useTransition } from 'react';
import { weekdayLabel } from '@/domain/menu/dates';
import { confirmPlan, generatePlan, rerollPlan, type PlanDay } from './actions';

export function PlanBuilder({ defaultStart, moods }: { defaultStart: string; moods: string[] }) {
  const [startDate, setStartDate] = useState(defaultStart);
  const [days, setDays] = useState(6);
  const [selected, setSelected] = useState<string[]>([]);
  const [menu, setMenu] = useState<PlanDay[] | null>(null);
  const [skips, setSkips] = useState<Record<number, number[]>>({});
  const [pending, startTransition] = useTransition();
  const options = { startDate, days, moods: selected };

  const toggleMood = (m: string) =>
    setSelected((s) => (s.includes(m) ? s.filter((x) => x !== m) : [...s, m]));

  const generate = () =>
    startTransition(async () => {
      setSkips({});
      setMenu(await generatePlan(options));
    });

  const reroll = (i: number) => {
    if (!menu) return;
    const skip = [...(skips[i] ?? []), menu[i].dishId];
    startTransition(async () => {
      setSkips((s) => ({ ...s, [i]: skip }));
      setMenu(await rerollPlan(menu, i, options, skip));
    });
  };

  const confirm = () => {
    if (!menu) return;
    startTransition(async () => {
      await confirmPlan(menu);
    });
  };

  return (
    <div className="space-y-4">
      <div className="flex gap-2">
        <label className="flex-1">開始日
          <input type="date" value={startDate} onChange={(e) => setStartDate(e.target.value)} className="w-full rounded border p-2" />
        </label>
        <label className="w-24">日数
          <select value={days} onChange={(e) => setDays(Number(e.target.value))} className="w-full rounded border p-2">
            {[1, 2, 3, 4, 5, 6, 7].map((n) => <option key={n} value={n}>{n}</option>)}
          </select>
        </label>
      </div>
      {moods.length > 0 && (
        <div>
          <p className="mb-1 text-sm">今週の気分</p>
          <div className="flex flex-wrap gap-2">
            {moods.map((m) => (
              <button key={m} type="button" onClick={() => toggleMood(m)}
                className={`rounded-full border px-3 py-1 text-sm ${selected.includes(m) ? 'border-orange-500 bg-orange-500 text-white' : ''}`}>
                {m}
              </button>
            ))}
          </div>
        </div>
      )}
      <button onClick={generate} disabled={pending} className="w-full rounded bg-orange-500 p-3 text-white disabled:opacity-50">
        献立を作る
      </button>
      {menu && (
        <>
          {menu.length < days && <p className="text-sm text-red-600">料理が足りません({menu.length}日分だけ作りました)</p>}
          <ul className="divide-y">
            {menu.map((m, i) => (
              <li key={m.date} className="flex items-center justify-between py-3">
                <span><span className="mr-2 text-sm text-gray-500">{m.date.slice(5)}({weekdayLabel(m.date)})</span>{m.dishName}</span>
                <button onClick={() => reroll(i)} disabled={pending} className="text-sm text-orange-600">別のにする</button>
              </li>
            ))}
          </ul>
          {menu.length > 0 && (
            <button onClick={confirm} disabled={pending} className="w-full rounded border border-orange-500 p-3 text-orange-600 disabled:opacity-50">
              この献立で確定
            </button>
          )}
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 4: ページを書く**

`src/app/plan/page.tsx`:

```tsx
import { getDishStats } from '@/db/repo';
import { listMoods } from '@/domain/dish-stats';
import { nextSunday, todayInTokyo } from '@/domain/menu/dates';
import { PlanBuilder } from './plan-builder';

export const dynamic = 'force-dynamic';

export default async function PlanPage() {
  const { dishes } = await getDishStats();
  return (
    <main className="mx-auto max-w-md p-4">
      <h1 className="mb-4 text-xl font-bold">献立を作る</h1>
      <PlanBuilder defaultStart={nextSunday(todayInTokyo())} moods={listMoods(dishes)} />
    </main>
  );
}
```

- [ ] **Step 5: 手動で確認する**

Run: `npm run dev` → `/plan`
Expected:
- 開始日が次の日曜日、日数が 6 になっている
- 「献立を作る」で日〜金の6日分が出て、平日に手間 4〜5 の料理が出にくい
- 気分のチップを選んで作り直すと、その主食材の料理が増える
- 「別のにする」を2回押しても、最初の料理に戻らない
- 「この献立で確定」を押すと `/` に移り、Supabase の `meals` に6行入っている

- [ ] **Step 6: 確認してコミット・PR**

```bash
npm test && npm run lint && npm run typecheck
git add -A
git commit -m "feat(plan): add menu planning screen" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push -u origin task/8-plan
gh pr create --fill --body "Closes #9

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

### Task 9: 今週の画面(評価・差し替え)(Issue #10)

**Files:**
- Modify: `src/app/page.tsx`(雛形を置き換える), `docs/superpowers/specs/2026-10-12-kondate-mvp-design.md`(§7)
- Create: `src/app/week-list.tsx`, `src/app/actions.ts`

**Interfaces:**
- Consumes: `listMealsBetween`, `listDishNames`, `setTaste`, `replaceMealDish`, `addDays`, `todayInTokyo`, `weekdayLabel`, `requireAuth`
- Produces: `setTasteAction(date: string, taste: number): Promise<void>` / `replaceDishAction(date: string, dishId: number): Promise<void>`

- [ ] **Step 1: ブランチを切る**

```bash
git switch main && git pull && git switch -c task/9-week
```

- [ ] **Step 2: Server Action を書く**

`src/app/actions.ts`:

```ts
'use server';

import { revalidatePath } from 'next/cache';
import { z } from 'zod';
import { replaceMealDish, setTaste } from '@/db/repo';
import { requireAuth } from '@/lib/require-auth';

const dateSchema = z.string().regex(/^\d{4}-\d{2}-\d{2}$/);

export async function setTasteAction(date: string, taste: number): Promise<void> {
  await requireAuth();
  await setTaste(dateSchema.parse(date), z.number().int().min(1).max(5).parse(taste));
  revalidatePath('/');
}

export async function replaceDishAction(date: string, dishId: number): Promise<void> {
  await requireAuth();
  await replaceMealDish(dateSchema.parse(date), z.number().int().positive().parse(dishId));
  revalidatePath('/');
}
```

- [ ] **Step 3: 一覧の部品を書く**

`src/app/week-list.tsx`:

```tsx
'use client';

import { useTransition } from 'react';
import { weekdayLabel } from '@/domain/menu/dates';
import { replaceDishAction, setTasteAction } from './actions';

type WeekMeal = { date: string; dishId: number; taste: number | null; dishName: string };

export function WeekList({ today, meals, dishes }: { today: string; meals: WeekMeal[]; dishes: { id: number; name: string }[] }) {
  const [pending, startTransition] = useTransition();
  return (
    <ul className="divide-y">
      {meals.map((m) => (
        <li key={m.date} className="py-3">
          <div className="flex items-center gap-2">
            <span className="w-16 text-sm text-gray-500">{m.date.slice(5)}({weekdayLabel(m.date)})</span>
            <select value={m.dishId} disabled={pending}
              onChange={(e) => startTransition(() => replaceDishAction(m.date, Number(e.target.value)))}
              className="flex-1 rounded border p-2">
              {dishes.map((d) => <option key={d.id} value={d.id}>{d.name}</option>)}
            </select>
          </div>
          {m.date <= today && (
            <div className="mt-2 flex gap-1 pl-16" aria-label="美味しさ">
              {[1, 2, 3, 4, 5].map((n) => (
                <button key={n} disabled={pending} aria-label={`${n}点`}
                  onClick={() => startTransition(() => setTasteAction(m.date, n))}
                  className={`text-3xl ${m.taste !== null && n <= m.taste ? 'text-orange-500' : 'text-gray-300'}`}>
                  ★
                </button>
              ))}
            </div>
          )}
        </li>
      ))}
    </ul>
  );
}
```

- [ ] **Step 4: ページを書く**

`src/app/page.tsx`:

```tsx
import Link from 'next/link';
import { listDishNames, listMealsBetween } from '@/db/repo';
import { addDays, todayInTokyo } from '@/domain/menu/dates';
import { WeekList } from './week-list';

export const dynamic = 'force-dynamic';

export default async function HomePage() {
  const today = todayInTokyo();
  // 評価し忘れた分も付けられるよう、2日前から表示する
  const [meals, dishes] = await Promise.all([listMealsBetween(addDays(today, -2), addDays(today, 6)), listDishNames()]);
  return (
    <main className="mx-auto max-w-md p-4">
      <h1 className="mb-4 text-xl font-bold">今週の献立</h1>
      {meals.length === 0 ? (
        <p>確定した献立がありません。<Link href="/plan" className="text-orange-600 underline">献立を作る</Link></p>
      ) : (
        <WeekList today={today} meals={meals} dishes={dishes} />
      )}
    </main>
  );
}
```

- [ ] **Step 5: 手動で確認する**

Run: `npm run dev` → `/`
Expected:
- Task 8 で確定した献立が並ぶ
- 今日以前の日だけ ★ が出て、タップすると色が付き、Supabase の `meals.taste` に入る
- 料理を差し替えると、その日の ★ が消える(評価が NULL に戻る)

- [ ] **Step 6: spec を合わせる**

`docs/superpowers/specs/2026-10-12-kondate-mvp-design.md` §7「今週」の「今日から7日分」を「2日前から6日後まで(評価し忘れた分も付けられるように)」に直す

- [ ] **Step 7: 確認してコミット・PR**

```bash
npm test && npm run lint && npm run typecheck
git add -A
git commit -m "feat(week): add weekly view with rating and dish swap" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push -u origin task/9-week
gh pr create --fill --body "Closes #10

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

### Task 10: PWA 化と本番公開・受け入れ確認(Issue #11)

**Files:**
- Create: `src/app/manifest.ts`, `src/app/apple-icon.tsx`
- Delete: 雛形の `public/*.svg`(どこからも参照されていなければ)

- [ ] **Step 1: ブランチを切る**

```bash
git switch main && git pull && git switch -c task/10-pwa
```

- [ ] **Step 2: manifest とアイコンを書く**

`src/app/manifest.ts`:

```ts
import type { MetadataRoute } from 'next';

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: '献立',
    short_name: '献立',
    start_url: '/',
    display: 'standalone',
    background_color: '#ffffff',
    theme_color: '#f97316',
    icons: [{ src: '/apple-icon', sizes: '180x180', type: 'image/png' }],
  };
}
```

`src/app/apple-icon.tsx`:

```tsx
import { ImageResponse } from 'next/og';

export const size = { width: 180, height: 180 };
export const contentType = 'image/png';

export default function AppleIcon() {
  return new ImageResponse(
    (
      <div style={{ width: '100%', height: '100%', display: 'flex', alignItems: 'center', justifyContent: 'center', background: '#f97316', fontSize: 120 }}>
        🍚
      </div>
    ),
    size,
  );
}
```

- [ ] **Step 3: ビルドを確認する**

Run: `npm run build`
Expected: 成功。ビルドのルート一覧に `/manifest.webmanifest` と `/apple-icon` が出る

- [ ] **Step 4: コミット・PR**

```bash
npm test && npm run lint && npm run typecheck
git add -A
git commit -m "feat(pwa): add manifest and apple icon" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push -u origin task/10-pwa
gh pr create --fill --body "Closes #11

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

- [ ] **Step 5: 🧑 Vercel に接続する**
  1. https://vercel.com に GitHub アカウントでログイン →「Add New Project」→ `pj8898h2/kondate-app` を Import
  2. Environment Variables に `DATABASE_URL` `APP_PASSWORD` `AUTH_SECRET` を、`.env.local` と同じ値で入れる(Production と Preview の両方)
  3. Deploy する。以降は PR ごとにプレビュー、main へのマージで本番に自動で出る

Expected: Task 10 の PR に Vercel のプレビュー URL が付き、そこでログインできる

- [ ] **Step 6: 🧑 受け入れ確認**(spec §9。PR をマージして本番に出てから行う)
  1. iPhone の Safari で本番 URL を開き、ログイン → 共有 →「ホーム画面に追加」。アイコンが 🍚 になり、全画面で開く
  2. 「献立を作る」で日〜金の6日分を作り、1日を「別のにする」で差し替えて確定する
  3. 翌日に「今週」で前日の夕食に ★ を付ける
  4. 翌週の献立を作り、その料理が出ない(または後ろに下がる)ことを確かめる

- [ ] **Step 7: vault を更新する**

`c:\Users\pj889\Mynotes\projects\private\kondate-app.md` に本番 URL と Slice 1 の完了日を追記する

---

## Self-Review メモ(計画作成時に確認済み)

- spec のカバー範囲(§3 Slice 1): 料理の登録・編集・一覧 = Task 7 / 一括投入 = Task 4・7 / 献立生成 = Task 2・3・8 / 差し替え・確定 = Task 3・8 / 評価 = Task 9 / パスワード = Task 6 / 公開・CI・PWA = Task 1・10
- §8 エラー処理: 入力チェック = Task 4・7 / パスワード違い = Task 6 / 料理が足りない = Task 3・8。DB 接続エラーは Next.js 既定のエラー画面に任せる(専用の `error.tsx` は作らない)
- spec からの変更点(Task 2 Step 9 と Task 9 Step 6 で spec に反映する): `dishes.baseTaste` の追加 / `rerollDay` の `skip` / 「次の日曜日」は今日が日曜なら今日 / 今週の表示は2日前から
