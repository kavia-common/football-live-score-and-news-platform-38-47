# MySQL Schema (football live scores platform)

This database schema is designed to support:
- User login
- Teams
- Matches (live + schedules)
- Standings (league table)
- Minimal supporting tables for personalization and match timeline

## Connection settings (normalized)

This DB container currently provisions MySQL with:

- Database: `myapp`
- App user: `appuser`
- App password: `dbuser123`
- Host: `localhost`
- Port: `5000`

**Important:** The overall work item runtime metadata shows the DB container exposed at port `5001`, but the DB container scripts and `db_connection.txt` are configured for `5000`. For reliable backend connectivity, the backend should use the same values as the DB container (`db_connection.txt` is authoritative in this container).

Environment variables expected by other containers/tools:
- `MYSQL_URL` (example: `mysql://localhost:5000/myapp`)
- `MYSQL_USER` (`appuser`)
- `MYSQL_PASSWORD` (`dbuser123`)
- `MYSQL_DB` (`myapp`)
- `MYSQL_PORT` (`5000`)

## Tables

### `users`
Stores platform users.

Columns:
- `id` BIGINT PK
- `email` unique
- `password_hash`
- `display_name`
- `created_at`, `updated_at`

Indexes:
- `uq_users_email` unique(email)

### `teams`
Stores teams, optionally mapped to API-Football team IDs.

Columns:
- `id` BIGINT PK
- `api_football_team_id` (unique, nullable)
- `name`
- `country`
- `logo_url`
- `founded`
- `venue_name`, `venue_city`
- `created_at`, `updated_at`

Indexes:
- `uq_teams_api_id` unique(api_football_team_id)
- `idx_teams_name` (name)

### `competitions`
Minimal leagues/competitions table for standings and match grouping.

Columns:
- `id` BIGINT PK
- `api_football_league_id` (nullable)
- `name`
- `country`
- `season`
- `logo_url`
- `created_at`, `updated_at`

Indexes:
- `uq_competitions_api_id_season` unique(api_football_league_id, season)
- `idx_competitions_name` (name)

### `matches`
Stores fixtures (live and scheduled) and key score/status fields.

Columns:
- `id` BIGINT PK
- `api_football_fixture_id` (unique, nullable)
- `competition_id` FK -> competitions(id) (nullable)
- `season`
- `round_name`
- status fields: `status_short`, `status_long`, `status_elapsed`
- `match_date_utc` (DATETIME, required)
- `timezone`
- `home_team_id` FK -> teams(id)
- `away_team_id` FK -> teams(id)
- score fields: `home_score`, `away_score`, HT/FT splits
- venue fields: `venue_name`, `venue_city`
- `created_at`, `updated_at`

Indexes:
- `uq_matches_api_fixture_id` unique(api_football_fixture_id)
- `idx_matches_date` (match_date_utc)
- `idx_matches_comp_season` (competition_id, season)
- `idx_matches_home_team_date` (home_team_id, match_date_utc)
- `idx_matches_away_team_date` (away_team_id, match_date_utc)

### `standings`
Stores league table rows per competition/season/team.

Columns:
- `id` BIGINT PK
- `competition_id` FK -> competitions(id)
- `season`
- `team_id` FK -> teams(id)
- `position`, `points`, `played`, `win`, `draw`, `loss`
- `goals_for`, `goals_against`, `goal_diff`
- `form`
- `updated_at`

Indexes:
- `uq_standings_comp_season_team` unique(competition_id, season, team_id)
- `idx_standings_comp_season_position` (competition_id, season, position)

### `user_favorite_teams` (supporting)
Many-to-many mapping for personalization.

Columns:
- `user_id` FK -> users(id)
- `team_id` FK -> teams(id)
- `created_at`

PK:
- (user_id, team_id)

### `match_events` (supporting)
Stores event timeline items for live matches.

Columns:
- `id` BIGINT PK
- `match_id` FK -> matches(id)
- `api_football_event_id` (unique, nullable)
- `elapsed`, `extra`
- `team_id` FK -> teams(id) (nullable)
- `player_name`, `assist_name`
- `event_type`, `event_detail`, `comments`
- `created_at`

Indexes:
- `uq_match_events_api` unique(api_football_event_id)
- `idx_match_events_match` (match_id)

## Notes
- All tables use `InnoDB`, `utf8mb4`, `utf8mb4_unicode_ci`.
- Foreign keys are designed to keep referential integrity while allowing upstream API IDs to be nullable until data is synced.
