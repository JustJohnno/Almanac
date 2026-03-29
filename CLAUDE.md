*** PROJECT OVERVIEW ***

┌─────────────────────────────────────────────────────────────────────────────┐
│                        RACING INTELLIGENCE PLATFORM                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│   │   SCRAPING  │    │    DATA     │    │  PROCESSING │    │     UI      │ │
│   │   ENGINE    │───▶│   PIPELINE  │───▶│   ENGINE    │───▶│   LAYER     │ │
│   └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘ │
│          │                  │                  │                  │          │
│          ▼                  ▼                  ▼                  ▼          │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        DATABASE LAYER                                │   │
│   │         Horses | Races | Meetings | Sectionals | Comments           │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘


-> INITIAL EXTERNAL SOURCES

┌─────────────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL DATA SOURCES                               │
├───────────────────────────────┬─────────────────────────────────────────────┤
│                               │                                              │
│   ┌───────────────────────┐   │   ┌─────────────────────────────────────┐   │
│   │   RACING AUSTRALIA    │   │   │           RACING.COM                │   │
│   │   (racingaustralia    │   │   │                                     │   │
│   │    .horse)            │   │   │                                     │   │
│   ├───────────────────────┤   │   ├─────────────────────────────────────┤   │
│   │ • Upcoming Fields     │   │   │ • Sectional Time Results            │   │
│   │ • Race Cards          │   │   │ • Split Time Data                   │   │
│   │ • Runner Lists        │   │   │ • Position at Sectional Points      │   │
│   │ • Race Results        │   │   │ • Race Distance Splits              │   │
│   │ • Meeting Details     │   │   │                                     │   │
│   └───────────────────────┘   │   └─────────────────────────────────────┘   │
│                               │                                              │
└───────────────────────────────┴─────────────────────────────────────────────┘


*** PROCESS A ***

This process is for upcoming meetings and races.

┌─────────────────────────────────────────────────────────────────────────────┐
│              PROCESS A: UPCOMING FIELDS SCRAPE & HORSE MATCHING              │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐
│    START     │
│  (Scheduled  │
│   or Manual  │
│   Trigger)   │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  STEP A1: SCRAPE RACING AUSTRALIA — UPCOMING FIELDS  │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Target Data Captured:                               │
│  ┌────────────────────────────────────────────────┐  │
│  │ • Meeting Date          • Venue / Track        │  │
│  │ • Race Number           • Race Name            │  │
│  │ • Distance              • Class / Conditions   │  │
│  │ • Runner Name           • Barrier Number       │  │
│  │ • Jockey                • Trainer              │  │
│  │ • Weight                • Runner Number        │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  Method: HTTP GET / HTML Parse / API Call            │
│  Frequency: Daily scheduled + on-demand refresh      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP A2: DATA NORMALISATION & STAGING               │
├──────────────────────────────────────────────────────┤
│                                                      │
│  • Strip whitespace / special characters             │
│  • Standardise horse names to UPPERCASE              │
│  • Normalise venue names to master venue list        │
│  • Parse & validate date formats                     │
│  • Assign internal Meeting_ID + Race_ID (staging)    │
│  • Write to STAGING TABLE for matching               │
│                                                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP A3: HORSE NAME MATCHING ENGINE                 │
├──────────────────────────────────────────────────────┤
│                                                      │
│  For each Runner in Staging:                         │
│                                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │  EXACT MATCH                                │    │
│  │  Query: SELECT * FROM horses                │    │
│  │         WHERE horse_name = [staged_name]    │    │
│  └────────────────────┬────────────────────────┘    │
│                       │                             │
│              ┌────────┴────────┐                    │
│           MATCH?            NO MATCH                │
│              │                 │                    │
│              ▼                 ▼                    │
│       ┌────────────┐   ┌───────────────────────┐   │
│       │ Flag as    │   │ FUZZY MATCH ATTEMPT   │   │
│       │ MATCHED    │   │ (Levenshtein / token  │   │
│       │ horse_id   │   │  ratio algorithm)     │   │
│       │ assigned   │   └───────────┬───────────┘   │
│       └────────────┘               │               │
│                           ┌────────┴────────┐      │
│                        MATCH?           NO MATCH   │
│                           │                 │      │
│                           ▼                 ▼      │
│                    ┌────────────┐   ┌────────────┐ │
│                    │ Flag for   │   │ CREATE NEW │ │
│                    │ USER       │   │ HORSE      │ │
│                    │ REVIEW &   │   │ RECORD     │ │
│                    │ CONFIRM    │   │ (stub)     │ │
│                    └────────────┘   └────────────┘ │
│                                                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP A4: RECALL & ASSEMBLE HORSE PROFILE DATA       │
├──────────────────────────────────────────────────────┤
│                                                      │
│  For each MATCHED horse_id, retrieve from DB:        │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  FROM: horses table                            │  │
│  │  • Horse name, age, sex, colour                │  │
│  │  • Trainer, owner                              │  │
│  │                                                │  │
│  │  FROM: comments table                          │  │
│  │  • All previously entered analyst comments     │  │
│  │  • Indexed by race_id + meeting_id             │  │
│  │  • Ordered by date DESC                        │  │
│  │                                                │  │
│  │  FROM: race_results table                      │  │
│  │  • Past finishing positions                    │  │
│  │  • Class, distance, track                      │  │
│  │                                                │  │
│  │  FROM: sectionals table                        │  │
│  │  • All stored sectional times                  │  │
│  │  • Calculated part-time metrics                │  │
│  │  • Par time comparisons                        │  │
│  │                                                │  │
│  │  FROM: custom_datapoints table                 │  │
│  │  • Any user-defined data fields                │  │
│  │  • Ratings, flags, notes                       │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP A5: RENDER UPCOMING FIELDS VIEW (UI)           │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Display per Race Card:                              │
│  ┌────────────────────────────────────────────────┐  │
│  │ Meeting Header: Date | Venue | Race No | Name  │  │
│  ├──────┬──────┬────────┬───────┬─────────────────┤  │
│  │ No.  │Horse │ Jockey │Weight │  STORED DATA    │  │
│  ├──────┼──────┼────────┼───────┼─────────────────┤  │
│  │  1   │ XXX  │  XXX   │  58   │ [Comments ▼]   │  │
│  │      │      │        │       │ [Sectionals ▼] │  │
│  │      │      │        │       │ [Data Points ▼]│  │
│  ├──────┼──────┼────────┼───────┼─────────────────┤  │
│  │  2   │ XXX  │  XXX   │  56   │ [Comments ▼]   │  │
│  │      │ ⚠️NEW │        │       │ No prior data   │  │
│  └──────┴──────┴────────┴───────┴─────────────────┘  │
│                                                      │
│  ⚠️ = New horse (no existing DB record)              │
│  Expandable panels per runner for historical data    │
│                                                      │
└──────────────────────────────────────────────────────┘


*** PROCESS B ***

This process involves completed races.


┌─────────────────────────────────────────────────────────────────────────────┐
│           PROCESS B: RACING.COM SECTIONALS — CSV BUILD & CALCULATIONS        │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐
│    START     │
│  (Post-Race  │
│   Trigger /  │
│   Scheduled) │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  STEP B1: SOURCE SECTIONAL DATA FROM RACING.COM      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Raw Data Fields Captured:                           │
│  ┌────────────────────────────────────────────────┐  │
│  │ • Race_ID / Meeting reference                  │  │
│  │ • Horse Name                                   │  │
│  │ • Finish Position                              │  │
│  │ • Total Race Time                              │  │
│  │ • Sectional Split Points (e.g. 200m, 400m,    │  │
│  │   600m, 800m, 1000m, 1200m etc.)               │  │
│  │ • Time at each sectional point                 │  │
│  │ • Position at each sectional point             │  │
│  │ • Margin at each sectional point               │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP B2: BUILD INITIAL LONG FORMAT CSV              │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Structure — One row per runner per race:            │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │ meeting_date | venue | race_no | race_name   │   │
│  │ distance | horse_name | finish_pos           │   │
│  │ total_time                                   │   │
│  │ sec_200m | sec_400m | sec_600m | sec_800m    │   │
│  │ sec_1000m | sec_1200m | ... [to distance]    │   │
│  │ pos_200m | pos_400m | pos_600m | ...         │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP B3: PART TIME & DERIVED CALCULATIONS ENGINE    │
├──────────────────────────────────────────────────────┤
│                                                      │
│  For each runner row, calculate and append:          │
│                                                      │
│  ┌──── PART TIME CALCULATIONS ─────────────────────┐ │
│  │                                                 │ │
│  │  LAST 200m Time:                                │ │
│  │  = total_time − sec_time_at_(distance−200m)     │ │
│  │                                                 │ │
│  │  LAST 400m Time:                                │ │
│  │  = total_time − sec_time_at_(distance−400m)     │ │
│  │                                                 │ │
│  │  LAST 600m Time:                                │ │
│  │  = total_time − sec_time_at_(distance−600m)     │ │
│  │                                                 │ │
│  │  FIRST 400m Time:                               │ │
│  │  = sec_time_at_400m                             │ │
│  │                                                 │ │
│  │  FIRST 600m Time:                               │ │
│  │  = sec_time_at_600m                             │ │
│  │                                                 │ │
│  │  MID RACE SECTIONAL (e.g. 400-800m split):      │ │
│  │  = sec_time_at_800m − sec_time_at_400m          │ │
│  │                                                 │ │
│  └─────────────────────────────────────────────────┘ │
│                                                      │
│  ┌──── PAR TIME CALCULATIONS ──────────────────────┐ │
│  │                                                 │ │
│  │  Race Par Time (by distance/class/track):       │ │
│  │  = AVG(total_time) for class/dist/track         │ │
│  │                                                 │ │
│  │  Runner vs Par:                                 │ │
│  │  = total_time − race_par_time                   │ │
│  │  [negative = faster than par]                   │ │
│  │                                                 │ │
│  │  Sectional Par per split point:                 │ │
│  │  = AVG(split_time) across field for that split  │ │
│  │                                                 │ │
│  │  Runner Sectional vs Sectional Par:             │ │
│  │  = runner_split_time − sectional_par_time       │ │
│  │                                                 │ │
│  └─────────────────────────────────────────────────┘ │
│                                                      │
│  ┌──── RANKING CALCULATIONS ───────────────────────┐ │
│  │                                                 │ │
│  │  • Best Last 200m in race (rank)                │ │
│  │  • Best Last 400m in race (rank)                │ │
│  │  • Fastest overall (rank)                       │ │
│  │  • Positions gained/lost (start vs finish)      │ │
│  │                                                 │ │
│  └─────────────────────────────────────────────────┘ │
│                                                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP B4: APPEND CALCULATED COLUMNS TO CSV           │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Final Long CSV Column Schema:                       │
│  ┌────────────────────────────────────────────────┐  │
│  │ [Original Scraped Fields]                      │  │
│  │ + last_200m  + last_400m  + last_600m          │  │
│  │ + first_400m + first_600m                      │  │
│  │ + mid_sec_400_800                              │  │
│  │ + par_time   + vs_par                          │  │
│  │ + sec_par_200m ... sec_par_Xm                  │  │
│  │ + vs_sec_par_200m ... vs_sec_par_Xm            │  │
│  │ + last200_rank + last400_rank                  │  │
│  │ + positions_gained                             │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  Output: /exports/sectionals_[date]_[venue].csv      │
│                                                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP B5: IMPORT CSV DATA TO DATABASE                │
├──────────────────────────────────────────────────────┤
│                                                      │
│  • Upsert rows to sectionals table                   │
│  • Index by horse_id + race_id + meeting_id          │
│  • Conflict resolution: update on duplicate key      │
│  • Log import summary (rows imported / errors)       │
│                                                      │
└──────────────────────────────────────────────────────┘


*** PROCESS FLOW C ***

This proces is a post-race review with data entry/

┌─────────────────────────────────────────────────────────────────────────────┐
│            PROCESS C: RESULTS SCRAPE, PRESENTATION & DATA CAPTURE           │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐
│    START     │
│  (Post-Race  │
│   Trigger /  │
│   On-Demand) │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  STEP C1: SCRAPE RESULTS — RACING AUSTRALIA          │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Target Data Fields:                                 │
│  ┌────────────────────────────────────────────────┐  │
│  │ • Meeting Date          • Venue                │  │
│  │ • Race Number           • Race Name            │  │
│  │ • Distance              • Track Condition      │  │
│  │ • Finish Position       • Runner Name          │  │
│  │ • Runner Number         • Barrier              │  │
│  │ • Jockey                • Trainer              │  │
│  │ • Weight Carried        • Margin               │  │
│  │ • Finishing Time        • Dividend / SP        │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP C2: NORMALISE & STORE RESULTS                  │
├──────────────────────────────────────────────────────┤
│                                                      │
│  • Normalise horse names (same rules as Process A)   │
│  • Match to horse_id in horses table                 │
│  • Generate/confirm race_id & meeting_id             │
│  • Upsert to race_results table                      │
│  • Link to any existing staging data (from Proc A)   │
│                                                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP C3: ASSEMBLE RESULTS DISPLAY PACKAGE           │
├──────────────────────────────────────────────────────┤
│                                                      │
│  For each runner in finishing order, JOIN:           │
│                                                      │
│  race_results                                        │
│    LEFT JOIN sectionals    ON horse_id + race_id     │
│    LEFT JOIN comments      ON horse_id + race_id     │
│    LEFT JOIN par_times     ON race_id                │
│    LEFT JOIN custom_data   ON horse_id + race_id     │
│                                                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP C4: RENDER RESULTS VIEW (UI)                   │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Meeting Header Banner:                              │
│  ┌────────────────────────────────────────────────┐  │
│  │  📅 Date | 📍 Venue | Race X — [Race Name]     │  │
│  │  Distance: Xm | Track: Good 4 | Time: X:XX.Xs  │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  Results Table — Ordered by Finish Position:         │
│  ┌───┬──────────────┬──────┬────────┬──────────────┐ │
│  │Pos│ Horse Name   │Margin│Jockey  │ Weight       │ │
│  ├───┼──────────────┼──────┼────────┼──────────────┤ │
│  │ 1 │ HORSE A      │  —   │ J.XXXX │  58.0        │ │
│  │ 2 │ HORSE B      │  0.8 │ J.XXXX │  56.5        │ │
│  │ 3 │ HORSE C      │  1.2 │ J.XXXX │  55.0        │ │
│  └───┴──────────────┴──────┴────────┴──────────────┘ │
│                                                      │
│  Per Runner — Expandable Detail Panel:               │
│  ┌────────────────────────────────────────────────┐  │
│  │  ▼ HORSE A [pos: 1]                            │  │
│  │  ┌─────────────────────────────────────────┐   │  │
│  │  │ SECTIONAL TIMES                         │   │  │
│  │  │ 200m: X.Xs | 400m: X.Xs | 600m: X.Xs   │   │  │
│  │  │ Last 200: X.Xs | Last 400: X.Xs         │   │  │
│  │  │ vs Par: −0.15s ✅                        │   │  │
│  │  └─────────────────────────────────────────┘   │  │
│  │  ┌─────────────────────────────────────────┐   │  │
│  │  │ PAR TIME METRICS                        │   │  │
│  │  │ Race Par: X:XX.Xs                       │   │  │
│  │  │ Runner Time: X:XX.Xs                    │   │  │
│  │  │ Variance: −0.15s (above par)            │   │  │
│  │  └─────────────────────────────────────────┘   │  │
│  │  ┌─────────────────────────────────────────┐   │  │
│  │  │ 📝 ANALYST COMMENT                      │   │  │
│  │  │ ┌───────────────────────────────────┐   │   │  │
│  │  │ │                                   │   │   │  │
│  │  │ │  [Free text entry field]          │   │   │  │
│  │  │ │                                   │   │   │  │
│  │  │ └───────────────────────────────────┘   │   │  │
│  │  │  [💾 Save Comment]  [🗑 Clear]           │   │  │
│  │  └─────────────────────────────────────────┘   │  │
│  │  ┌─────────────────────────────────────────┐   │  │
│  │  │ PRIOR COMMENTS (most recent first)      │   │  │
│  │  │ 📅 DD/MM/YY — [Race] — "Comment text"  │   │  │
│  │  │ 📅 DD/MM/YY — [Race] — "Comment text"  │   │  │
│  │  └─────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│  STEP C5: USER DATA ENTRY & SAVE                     │
├──────────────────────────────────────────────────────┤
│                                                      │
│  User Actions Available:                             │
│  ┌────────────────────────────────────────────────┐  │
│  │  [1] Enter free text comment per runner        │  │
│  │  [2] Edit / confirm sectional time display     │  │
│  │  [3] View / edit par time metrics              │  │
│  │  [4] Enter custom data point values            │  │
│  │  [5] Flag runner (e.g. watch / remove)         │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  On SAVE:                                            │
│  ┌────────────────────────────────────────────────┐  │
│  │  Payload constructed:                          │  │
│  │  {                                             │  │
│  │    horse_id: [id],                             │  │
│  │    race_id: [id],                              │  │
│  │    meeting_id: [id],                           │  │
│  │    comment_text: "[user input]",               │  │
│  │    comment_date: [timestamp],                  │  │
│  │    custom_fields: { key: value, ... },         │  │
│  │    analyst_id: [user_id]                       │  │
│  │  }                                             │  │
│  │                                                │  │
│  │  → POST to API endpoint                        │  │
│  │  → Upsert to comments table                    │  │
│  │  → Upsert to custom_datapoints table           │  │
│  │  → Return success / error response             │  │
│  │  → UI confirmation toast message               │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘

*** GENERAL SUMMARY ***

┌─────────────────────────────────────────────────────────────────────────────┐
│                      END-TO-END SYSTEM FLOW SUMMARY                          │
└─────────────────────────────────────────────────────────────────────────────┘

 RACING AUSTRALIA                RACING.COM
       │                              │
       │ Scrape Fields & Results      │ Scrape Sectional Data
       ▼                              ▼
 ┌───────────────┐           ┌──────────────────┐
 │ Scraping      │           │ Sectional Data   │
 │ Engine        │           │ Collector        │
 └───────┬───────┘           └────────┬─────────┘
         │                            │
         ▼                            ▼
 ┌───────────────┐           ┌──────────────────┐
 │ Normalisation │           │ CSV Builder &    │
 │ & Staging     │           │ Calc Engine      │
 └───────┬───────┘           └────────┬─────────┘
         │                            │
         ▼                            │
 ┌───────────────┐                    │
 │ Horse Name    │                    │
 │ Matching      │                    │
 │ Engine        │                    │
 └───────┬───────┘                    │
         │                            │
         └──────────────┬─────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │                  │
              │   DATABASE       │
              │                  │
              │ horses           │
              │ meetings         │
              │ races            │
              │ race_results     │
              │ sectionals       │
              │ comments         │
              │ custom_datapoints│
              │                  │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │   API LAYER      │
              │  (REST / GraphQL)│
              └────────┬─────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
  ┌─────────────┐ ┌─────────┐ ┌──────────────┐
  │  UPCOMING   │ │RESULTS  │ │  SECTIONALS  │
  │  FIELDS     │ │  VIEW   │ │  / CSV       │
  │  VIEW       │ │         │ │  EXPORT      │
  │             │ │Comment  │ │              │
  │ Horse       │ │Entry    │ │              │
  │ History     │ │Fields   │ │              │
  │ Display     │ │         │ │              │
  └─────────────┘ └────┬────┘ └──────────────┘
                       │
                       ▼
              ┌──────────────────┐
              │  SAVE / UPDATE   │
              │  DATABASE        │
              │  (comments,      │
              │   datapoints)    │
              └──────────────────┘




┌─────────────────────────────────────────────────────────────────────────────┐
│                        DESIGN & TECHNICAL NOTES                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  SCRAPING RESILIENCE                                                          │
│  ├── Implement retry logic with exponential backoff                           │
│  ├── Store raw HTML/JSON snapshots before parsing                             │
│  ├── Alert on structural changes to source pages                              │
│  └── Respect robots.txt / rate limiting on target sites                      │
│                                                                               │
│  HORSE NAME MATCHING                                                          │
│  ├── Maintain a name alias table for known variations                         │
│  ├── Log all fuzzy matches for analyst review queue                           │
│  └── Confidence score threshold configurable (e.g. >85%)                     │
│                                                                               │
│  SECTIONAL CALCULATIONS                                                       │
│  ├── Handle variable distance races (split points differ)                     │
│  ├── NULL-safe calculations where splits not available                        │
│  └── Par time recalculates as dataset grows                                   │
│                                                                               │
│  DATA INTEGRITY                                                               │
│  ├── All saves timestamped with analyst_id                                    │
│  ├── Comment history preserved (no hard deletes)                              │
│  ├── Race/meeting IDs immutable once created                                  │
│  └── Audit log table for all write operations                                 │
│                                                                               │
│  UI / UX                                                                      │
│  ├── Expandable runner panels (not full page loads)                           │
│  ├── Auto-save draft comments (localStorage)                                  │
│  ├── Visual indicators for vs-par metrics (colour coded)                      │
│  └── Filter/sort results view by sectional metrics                            │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘