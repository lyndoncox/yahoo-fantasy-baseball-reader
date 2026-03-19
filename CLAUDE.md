# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install          # Install dependencies
npm start            # Run the main data fetch (writes allMyData.json)
npm run league-prefix  # Fetch and print the current MLB game key prefix
```

There are no test or lint scripts defined in package.json (ESLint config exists but no script).

## Configuration

Before running, copy `config.json.example` to `config.json` and populate:
- `CONSUMER_KEY` / `CONSUMER_SECRET`: From Yahoo Developer app
- `YAHOO_AUTH_CODE`: One-time authorization code from Yahoo OAuth flow
- `LEAGUE_KEY`: Format `{prefix}.l.{leagueId}` (e.g. `458.l.123456`). Run `npm run league-prefix` to get the current year's prefix.
- `TEAM`: Your team number (found in the Yahoo Fantasy Baseball URL)
- `AUTH_FILE`: Path to store the OAuth token (default `./credentials.json`)
- `FREE_AGENTS`: Number of pages of free agents to fetch (25 per page; `0` = first 25)

`config.json` and `credentials.json` are gitignored.

## Architecture

The project has two source files:

**`src/yahooFantasyBaseball.js`** — Core module exported as `exports.yfbb`. A single object containing:
- OAuth state (`CREDENTIALS`, `AUTH_HEADER`, `WEEK`)
- URL builder methods (`freeAgents()`, `myTeam()`, `scoreboard()`, etc.) that compose URLs from `config.json` values
- Auth methods: `readCredentials()` reads/creates the token file; `getInitialAuthorization()` exchanges the auth code for tokens; `refreshAuthorizationToken()` refreshes on expiry
- `makeAPIrequest(url)`: Central HTTP method using axios. Parses Yahoo's XML responses via `fast-xml-parser` into JSON. Auto-refreshes expired tokens.
- Data fetch methods (`getFreeAgents`, `getMyPlayers`, `getMyPlayersStats`, `getMyScoreboard`, `getStatsIDs`, `getCurrentRoster`, `getTransactions`, etc.) each call `makeAPIrequest` and extract the relevant nested path from Yahoo's `fantasy_content` response envelope.

**`src/index.js`** — Entry point. Calls each data fetch method in sequence, assembles results into a single object, and writes it to `allMyData.json`.

**`src/getLeaguePrefix.js`** — Standalone script that only calls `getLeaguePrefix()` to print the current MLB game key.

### OAuth flow
1. First run: no `AUTH_FILE` → calls `getInitialAuthorization()` using `YAHOO_AUTH_CODE` → saves token to `AUTH_FILE`
2. Subsequent runs: reads token from `AUTH_FILE`
3. On API call: if Yahoo returns `token_expired`, `makeAPIrequest` transparently refreshes and retries

### Yahoo API response shape
All responses are XML parsed to JSON. Data is always nested under `fantasy_content` (e.g. `result.fantasy_content.league.players.player`).
