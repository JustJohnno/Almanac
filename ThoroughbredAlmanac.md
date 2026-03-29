┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATABASE ENTITY MAP                                │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐         ┌──────────────────┐
│     HORSES       │         │    MEETINGS      │
├──────────────────┤         ├──────────────────┤
│ horse_id (PK)    │         │ meeting_id (PK)  │
│ horse_name       │         │ meeting_date     │
│ normalised_name  │         │ venue            │
│ sex              │         │ track_condition  │
│ colour           │         │ state            │
│ trainer          │         └────────┬─────────┘
│ owner            │                  │
│ created_at       │                  │
└────────┬─────────┘                  │
         │                            │
         │              ┌─────────────▼────────┐
         │              │       RACES          │
         │              ├──────────────────────┤
         │              │ race_id (PK)          │
         │              │ meeting_id (FK)       │
         │              │ race_number           │
         │              │ race_name             │
         │              │ distance              │
         │              │ class                 │
         │              │ par_time              │
         │              └──────────┬───────────┘
         │                         │
         │    ┌────────────────────┼────────────────────┐
         │    │                    │                    │
         ▼    ▼                    ▼                    ▼
┌────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  RACE_RESULTS  │   │   SECTIONALS     │   │    COMMENTS      │
├────────────────┤   ├──────────────────┤   ├──────────────────┤
│ result_id (PK) │   │ sectional_id(PK) │   │ comment_id (PK)  │
│ race_id (FK)   │   │ race_id (FK)     │   │ horse_id (FK)    │
│ horse_id (FK)  │   │ horse_id (FK)    │   │ race_id (FK)     │
│ meeting_id(FK) │   │ meeting_id (FK)  │   │ meeting_id (FK)  │
│ finish_pos     │   │ split_200m       │   │ comment_text     │
│ margin         │   │ split_400m       │   │ comment_date     │
│ finish_time    │   │ split_600m       │   │ analyst_id       │
│ jockey         │   │ split_800m       │   │ created_at       │
│ weight         │   │ split_Xm ...     │   │ updated_at       │
│ barrier        │   │ last_200m        │   └──────────────────┘
│ starting_price │   │ last_400m        │
└────────────────┘   │ last_600m        │   ┌──────────────────┐
                     │ first_400m       │   │ CUSTOM_DATAPOINTS│
                     │ vs_par           │   ├──────────────────┤
                     │ positions_gained │   │ datapoint_id(PK) │
                     │ last200_rank     │   │ horse_id (FK)    │
                     │ last400_rank     │   │ race_id (FK)     │
                     └──────────────────┘   │ field_key        │
                                            │ field_value      │
                                            │ created_at       │
                                            └──────────────────┘