# F1 Analysis Plan

Oct 7, 2026 · @Araav Rotella

Week 3 practice project using the Formula 1 World Championship dataset (1950 to 2026), following the five-role planning process. Main question: how well does starting grid position predict where a driver finishes, and has that changed across eras?

## Dataset

Source: [Arths17/f1-dataset](https://github.com/Arths17/f1-dataset), an extended version of the Kaggle/Ergast F1 dataset. CC0 license. It covers 1,165 races from 1950 to the 2026 season so far.

The data is already normalized (one table per observational unit, as in the Tidy Data paper), so most of the work is joining tables, not reshaping them.

| Table | One row per | Rows | Used for |
| --- | --- | --- | --- |
| `results.csv` | driver per race | 27,590 | grid, finishing position, status |
| `races.csv` | race | 1,165 | year, race name, date |
| `status.csv` | finish status code | 141 | Finished, +1 Lap, Engine, Did not qualify... |
| `drivers.csv` | driver | 879 | names |
| `constructors.csv` | team | 214 | team names |
| `qualifying.csv` | driver per race | 11,298 | Q1/Q2/Q3 times (1994 onward) |
| `pit_stops.csv` | pit stop | 12,783 | stop durations (2011 onward) |

IDs link everything: `results.raceId` to `races`, `results.statusId` to `status`, and so on. Missing values are written as the text `\N`, so every file must be read with `na_values="\\N"`.

## Step 1: Project director, high-level questions

Each question needs at least one join and a group-by. None can be answered with one Excel chart.

1. **Does starting position decide the race?** How well does grid position predict finishing position, and has that changed from the 1950s to today?
2. **Who beat their car?** Which drivers most outperformed their teammate in the same car, across qualifying and race results?
3. **Do fast pit stops win positions?** Do drivers with faster average pit stops gain more places from grid to finish?
4. **Where does overtaking happen?** Which circuits see the most position changes between grid and finish?
5. **How reliable are the cars?** How has the rate of mechanical retirements (engine, gearbox, hydraulics) changed by decade?

## Step 2: Project manager, sharpening question 1

**Chosen question:** How well does grid position predict finishing position, and has that changed across eras?

| Vague term | Decision | Why |
| --- | --- | --- |
| "Starting position" | `grid` from results. Exclude `grid = 0`. | 0 means a pit lane start or did not qualify, not a real grid slot. |
| "Finishing position" | `position`, classified finishers only. Drivers listed "+1 Lap" count as classified. | Retirements have no real finishing position. Reliability is measured separately. |
| "Predict" | Three measures: (a) Spearman rank correlation of grid vs finish per race, (b) % of wins from pole, (c) average places gained or lost. | One number hides too much. These cover the whole field, the front, and movement. |
| "Era" | Decade of `year` (1950s to 2020s). | Simple and even-sized. Regulation eras are a possible follow-up. |
| "Race" | Championship Grand Prix only. Exclude the Indianapolis 500 (1950 to 1960) and sprints. | The Indy 500 counted for points but had a separate field of drivers. Sprints are a different format. |

**Core variables (5):** `results.grid`, `results.position`, `results.statusId` to `status.status`, `races.year`, `races.name`.

**Out of scope:** weather, tyres, and team strength. They matter, but bring in other tables and other questions.

## Step 3: Data analyst, cleaning issues and plan

**What a first inspection found** (only `head()`, `value_counts()` and `isna()`, no analysis code):

| Issue | Size | Handling |
| --- | --- | --- |
| Missing values stored as text `\N` | every table | read with `na_values="\\N"` so numeric columns parse as numbers |
| `grid = 0` | 1,638 rows, mostly 1960s to 1990s | drop. Mostly "Did not qualify" (1,013), "Did not prequalify" (331) and "Withdrew" (164) |
| `position` missing | 11,077 rows (40%) | these are retirements and non-starters. Keep them for the reliability check, drop them for the correlation |
| Indianapolis 500 | 11 races, 1950 to 1960 | drop by race name |
| Same driver twice in one race | 91 rows, 1950 to 1978 | shared cars were allowed then. Keep the driver's best result per race |
| 2020s decade is incomplete | 2020 to 2026 so far | fine for averages, but label it as partial |

**Analysis plan (pseudocode)**

1. Load `results`, `races`, `status` with `\N` as missing.
2. Merge `races[[raceId, year, name]]` and `status` onto `results`. Check row count stays 27,590.
3. Add `decade = year // 10 * 10`.
4. Drop Indianapolis 500 races. Drop rows where status is "Did not qualify", "Did not prequalify" or "Withdrew" (they never started).
5. Build `starters`: rows with `grid > 0`. Add `classified = position.notna()`.
6. Build `finishers`: starters that are classified. Keep one row per (raceId, driverId), the best `position`.
7. Add `places_gained = grid - position` on finishers.
8. Per race: Spearman correlation of grid vs position (races with 5 or more finishers). Then average by decade.
9. Per decade: % of winners who started on pole, mean `places_gained`, and % of starters classified.
10. Plot each measure by decade.

**Reusable function (homework bonus).** Write `by_group(df, group_col, metric, agg="mean")` so the same summary works by decade, circuit, constructor or driver. Write `race_correlation(df)` once and reuse it for qualifying vs finish later.

## Step 4: Back-and-forth, feasibility

Question 1 is the cheapest and fully feasible. Questions 4 and 2 reuse most of its cleaning, so they are the best next additions.

| Question | Extra work beyond Q1 | Estimate | Blocker or risk |
| --- | --- | --- | --- |
| 1. Grid predicts finish | none, this is the base | 2 h | none |
| 4. Overtaking by circuit | join `circuits`, group by circuit instead of decade | +30 min | circuits with few races give noisy averages, so require 10+ races |
| 2. Beating your teammate | self-join results on (raceId, constructorId) | +1.5 h | 1950s teams ran 3 to 5 cars. Restrict to 1994 onward, when qualifying data starts and teams ran 2 cars |
| 5. Reliability by decade | map 141 status codes into Finished / Mechanical / Accident / Other | +2 h | the mapping is manual and a judgement call; document it |
| 3. Pit stops vs places gained | merge pit stops, parse durations | +3 h | data only from 2011. 587 durations are stored as `m:ss.sss` (red-flag stops) and need parsing or removal. Stop speed is confounded with strategy |

**Recommendation to the director:** answer 1 now, add 4 and 2 next (about 2 more hours). Postpone 3, since it costs the most and the answer will be the weakest, because pit stop speed and strategy are tangled together.

## Step 5: Programmer, checkpoints and results

The notebook `f1_grid_analysis.ipynb` follows the plan above. Each checkpoint prints numbers to verify before moving on.

| Checkpoint | Expected | Actual |
| --- | --- | --- |
| 1. After merges | 27,590 rows, 0 unmatched | 27,590 rows, 0 unmatched |
| 2. After cleaning | Indy 500 gone, only real starters left | 25,441 starters, 16,214 finishers, 1,154 races, 0 Indy rows |
| 3. Summary by decade | 8 rows, 1950s to 2020s | 8 rows |
| 4. Robustness check | compare decades among races where 75%+ finished | only possible from 2000 on (see below) |

**First results**

- **Pole matters more than ever.** 28% of 1980s winners started on pole, versus 56% in the 2020s so far.
- **The full grid order changed less.** Average grid vs finish correlation stays between 0.63 and 0.78 in every decade.
- **Reliability is the biggest change.** 48% of starters were classified in the 1980s, 87% now. Finishers moved an average of 6.2 places in the 1980s and 2.9 in the 2020s.
- **Limitation.** Races where 75%+ of the field finished barely exist before 2000, so this data can't fully separate reliability from car performance.

**Next:** add question 4 (circuits, already previewed in the notebook) and question 2 (teammates), as recommended in step 4.
