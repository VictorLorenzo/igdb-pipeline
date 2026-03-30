# 🎮 IGDB Silver Layer — Entity Relationship Diagram

> Schema: `games_analytics_silver` · Source: [IGDB API](https://api-docs.igdb.com/)

This document maps all Silver layer tables and their relationships. Diagrams use [Mermaid](https://mermaid.js.org/) and render natively on GitHub.

**Legend:**
- `PK` = Primary Key
- `FK` = Foreign Key (scalar bigint → lookup table)
- `array(bigint)` = Many-to-many relationship (requires `LATERAL VIEW explode()` in Spark SQL)
- Solid lines = direct FK joins
- `||--o{` = one-to-many · `}o--||` = many-to-one

---

## 📌 Overview

The `games` table is the **central hub** of the schema. Most of its columns are either scalar FKs to dimension/lookup tables or `array(bigint)` fields referencing related entities. The diagrams below are grouped by domain cluster for readability.

---

## 1. Core game entity

The `games` table connects to almost every other table in the schema — either through scalar FK columns or array columns that encode many-to-many relationships.

```mermaid
erDiagram
    games {
        bigint id PK
        varchar name
        varchar slug
        varchar summary
        varchar storyline
        bigint game_type FK
        bigint game_status FK
        bigint franchise FK
        bigint cover FK
        bigint parent_game FK
        bigint first_release_date
        double rating
        bigint rating_count
        double aggregated_rating
        double total_rating
        bigint hypes
    }

    game_types {
        bigint id PK
        varchar type
    }

    game_statuses {
        bigint id PK
        varchar status
    }

    genres {
        bigint id PK
        varchar name
        varchar slug
    }

    themes {
        bigint id PK
        varchar name
        varchar slug
    }

    game_modes {
        bigint id PK
        varchar name
    }

    player_perspectives {
        bigint id PK
        varchar name
    }

    keywords {
        bigint id PK
        varchar name
    }

    game_engines {
        bigint id PK
        varchar name
        varchar description
    }

    games }o--|| game_types : "game_type → id"
    games }o--|| game_statuses : "game_status → id"
    games }o--o{ genres : "genres[] → id (explode)"
    games }o--o{ themes : "themes[] → id (explode)"
    games }o--o{ game_modes : "game_modes[] → id (explode)"
    games }o--o{ player_perspectives : "player_perspectives[] → id (explode)"
    games }o--o{ keywords : "keywords[] → id (explode)"
    games }o--o{ game_engines : "game_engines[] → id (explode)"
```

### Array fields on `games` (many-to-many, resolved via `explode()`)

| Column | Target Table | Join |
|--------|-------------|------|
| `genres[]` | `genres` | `explode(g.genres) → genres.id` |
| `themes[]` | `themes` | `explode(g.themes) → themes.id` |
| `platforms[]` | `platforms` | `explode(g.platforms) → platforms.id` |
| `game_modes[]` | `game_modes` | `explode(g.game_modes) → game_modes.id` |
| `player_perspectives[]` | `player_perspectives` | `explode(g.player_perspectives) → player_perspectives.id` |
| `keywords[]` | `keywords` | `explode(g.keywords) → keywords.id` |
| `game_engines[]` | `game_engines` | `explode(g.game_engines) → game_engines.id` |
| `age_ratings[]` | `age_ratings` | `explode(g.age_ratings) → age_ratings.id` |
| `involved_companies[]` | `involved_companies` | `explode(g.involved_companies) → involved_companies.id` |
| `release_dates[]` | `release_dates` | `explode(g.release_dates) → release_dates.id` |
| `language_supports[]` | `language_supports` | `explode(g.language_supports) → language_supports.id` |
| `multiplayer_modes[]` | `multiplayer_modes` | `explode(g.multiplayer_modes) → multiplayer_modes.id` |
| `dlcs[]` | `games` | self-referencing |
| `expansions[]` | `games` | self-referencing |
| `remakes[]` | `games` | self-referencing |
| `remasters[]` | `games` | self-referencing |
| `similar_games[]` | `games` | self-referencing |
| `franchises[]` | `franchises` | `explode(g.franchises) → franchises.id` |
| `collections[]` | `collections` | `explode(g.collections) → collections.id` |

---

## 2. Companies and involvement

The `involved_companies` table is the **bridge** between `games` and `companies`, with boolean flags indicating each company's role.

```mermaid
erDiagram
    involved_companies {
        bigint id PK
        bigint company FK
        bigint game FK
        boolean developer
        boolean publisher
        boolean porting
        boolean supporting
    }

    companies {
        bigint id PK
        varchar name
        varchar description
        bigint status FK
        bigint country
        bigint parent FK
    }

    company_statuses {
        bigint id PK
        varchar name
    }

    company_logos {
        bigint id PK
    }

    games ||--o{ involved_companies : "game ← involved_companies.game"
    involved_companies }o--|| companies : "company → companies.id"
    companies }o--|| company_statuses : "status → id"
    companies }o--o| companies : "parent → id (self-ref)"
```

---

## 3. Platforms

Platform hierarchy with type and family dimension tables.

```mermaid
erDiagram
    platforms {
        bigint id PK
        varchar name
        varchar abbreviation
        bigint platform_type FK
        bigint platform_family FK
        bigint generation
    }

    platform_types {
        bigint id PK
        varchar name
    }

    platform_families {
        bigint id PK
        varchar name
    }

    platforms }o--|| platform_types : "platform_type → id"
    platforms }o--|| platform_families : "platform_family → id"
    games }o--o{ platforms : "platforms[] → id (explode)"
```

---

## 4. Release dates

Per-game, per-platform, per-region release information.

```mermaid
erDiagram
    release_dates {
        bigint id PK
        bigint game FK
        bigint platform FK
        bigint release_region FK
        bigint date
        varchar human
        bigint y
        bigint m
        bigint status FK
        bigint date_format FK
    }

    regions {
        bigint id PK
        varchar name
    }

    release_date_statuses {
        bigint id PK
        varchar name
    }

    date_formats {
        bigint id PK
        varchar name
    }

    games ||--o{ release_dates : "game"
    release_dates }o--|| platforms : "platform → id"
    release_dates }o--|| regions : "release_region → id"
    release_dates }o--|| release_date_statuses : "status → id"
    release_dates }o--|| date_formats : "date_format → id"
```

---

## 5. Age ratings

Content rating per organization (ESRB, PEGI, CERO, etc.).

```mermaid
erDiagram
    age_ratings {
        bigint id PK
        bigint organization FK
        bigint rating_category FK
        varchar synopsis
    }

    age_rating_organizations {
        bigint id PK
        varchar name
    }

    age_rating_categories {
        bigint id PK
        varchar rating
        bigint organization FK
    }

    age_rating_content_descriptions_v2 {
        bigint id PK
    }

    games }o--o{ age_ratings : "age_ratings[] → id (explode)"
    age_ratings }o--|| age_rating_organizations : "organization → id"
    age_ratings }o--|| age_rating_categories : "rating_category → id"
```

---

## 6. Localization

Language support per game, qualified by support type (audio, subtitles, interface).

```mermaid
erDiagram
    language_supports {
        bigint id PK
        bigint game FK
        bigint language FK
        bigint language_support_type FK
    }

    languages {
        bigint id PK
        varchar name
        varchar native_name
        varchar locale
    }

    language_support_types {
        bigint id PK
        varchar name
    }

    games ||--o{ language_supports : "game"
    language_supports }o--|| languages : "language → id"
    language_supports }o--|| language_support_types : "language_support_type → id"
```

---

## 7. Multiplayer, time-to-beat and popularity

Gameplay metadata: multiplayer capabilities, completion times, and popularity metrics.

```mermaid
erDiagram
    multiplayer_modes {
        bigint id PK
        bigint game FK
        bigint platform FK
        boolean onlinecoop
        boolean offlinecoop
        boolean lancoop
        boolean campaigncoop
        boolean splitscreen
        boolean dropin
        bigint onlinemax
        bigint offlinemax
    }

    game_time_to_beats {
        bigint id PK
        bigint game_id FK
        bigint hastily
        bigint normally
        bigint completely
        bigint count
    }

    popularity_primitives {
        bigint id PK
        bigint game_id FK
        bigint popularity_type FK
        bigint popularity_source
        double value
    }

    popularity_types {
        bigint id PK
        varchar name
        bigint popularity_source
    }

    games ||--o{ multiplayer_modes : "game"
    multiplayer_modes }o--|| platforms : "platform → id"
    games ||--o{ game_time_to_beats : "game_id"
    games ||--o{ popularity_primitives : "game_id"
    popularity_primitives }o--|| popularity_types : "popularity_type → id"
```

---

## 8. Media and content

Child tables for visual assets, alternative names, external links, and website references.

```mermaid
erDiagram
    covers {
        bigint id PK
        bigint game FK
        varchar url
    }

    screenshots {
        bigint id PK
        bigint game FK
        varchar url
    }

    game_videos {
        bigint id PK
        bigint game FK
        varchar video_id
    }

    artworks {
        bigint id PK
        bigint game FK
    }

    alternative_names {
        bigint id PK
        bigint game FK
        varchar name
    }

    external_games {
        bigint id PK
        bigint game FK
        bigint category FK
    }

    external_game_sources {
        bigint id PK
        varchar name
    }

    websites {
        bigint id PK
        bigint game FK
        bigint category FK
        varchar url
    }

    website_types {
        bigint id PK
        varchar type
    }

    games ||--o{ covers : "game"
    games ||--o{ screenshots : "game"
    games ||--o{ game_videos : "game"
    games ||--o{ artworks : "game"
    games ||--o{ alternative_names : "game"
    games ||--o{ external_games : "game"
    external_games }o--|| external_game_sources : "category → id"
    games ||--o{ websites : "game"
    websites }o--|| website_types : "category → id"
```

---

## 9. Collections, franchises, characters and events

Grouping entities: franchises own game arrays, collections use a membership bridge table, characters and events link to games via arrays.

```mermaid
erDiagram
    franchises {
        bigint id PK
        varchar name
    }

    collections {
        bigint id PK
        varchar name
        varchar slug
    }

    collection_memberships {
        bigint id PK
        bigint game FK
        bigint collection FK
        bigint type FK
    }

    collection_membership_types {
        bigint id PK
        varchar name
    }

    characters {
        bigint id PK
        varchar name
        bigint gender FK
        bigint species FK
    }

    character_genders {
        bigint id PK
        varchar name
    }

    character_species {
        bigint id PK
        varchar name
    }

    events {
        bigint id PK
        varchar name
        varchar description
        bigint start_time
        bigint end_time
    }

    game_localizations {
        bigint id PK
        bigint game FK
        bigint region FK
    }

    games }o--o{ franchises : "franchises[] → id (explode)"
    games ||--o{ collection_memberships : "game"
    collection_memberships }o--|| collections : "collection → id"
    collection_memberships }o--|| collection_membership_types : "type → id"
    characters }o--|| character_genders : "gender → id"
    characters }o--|| character_species : "species → id"
    characters }o--o{ games : "games[] → id (explode)"
    events }o--o{ games : "games[] → id (explode)"
    games ||--o{ game_localizations : "game"
    game_localizations }o--|| regions : "region → id"
```

---

## 📊 Table count summary

| Domain | Tables | Role |
|--------|--------|------|
| Core game entity | 9 | Hub + lookup dimensions |
| Companies | 4 | Bridge + company master |
| Platforms | 3 | Hierarchy with type/family |
| Release dates | 4 | Per-game per-platform releases |
| Age ratings | 3 | Content rating per org |
| Localization | 3 | Language support matrix |
| Multiplayer & gameplay | 4 | Co-op, time-to-beat, popularity |
| Media & content | 8 | Assets, links, external refs |
| Collections & groups | 9 | Franchises, characters, events |
| **Total** | **47** | |

---

## 🔧 Spark SQL notes

Since IGDB stores many-to-many relationships as `array(bigint)` columns on the `games` table, you must use `LATERAL VIEW explode()` to flatten them before joining:

```sql
-- Example: resolve genre names for each game
SELECT g.id, g.name, ge.name AS genre_name
FROM games_analytics_silver.games g
LATERAL VIEW explode(g.genres) AS genre_id
JOIN games_analytics_silver.genres ge ON ge.id = genre_id
```

> ⚠️ In Spark SQL, you **cannot** place a `JOIN` immediately after a `LATERAL VIEW`. Wrap the explode in a subquery first, then join the lookup table to the subquery result.