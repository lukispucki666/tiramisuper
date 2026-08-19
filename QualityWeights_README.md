# Make quality weights configurable (Movie + TV sync)

## What changed

- `internal/config/config.go`
  - Replaced the unused `QualityWeights`/`TVQualityWeights` structs with `MovieWeights` and
    `TVWeights`, which map 1:1 to the scoring fields actually used by the Movie/TV engines.
  - Both types have a custom `UnmarshalJSON` that first applies the defaults
    (`DefaultMovieWeights()` / `DefaultTVWeights()`) and then overwrites only the fields present
    in the JSON. Omitting a field keeps its default value.
  - New `disable_4k` (bool) field on both structs.
  - New `seeder_weight` / `seeder_cap` (int) fields on both structs: the seeder contribution to
    the score is `min(seeders, seeder_cap) * seeder_weight`.

- `internal/syncer/engines/movie_go.go`
  - Removed the constant block (`mMovie4KBase`, `mMovieHDRBonus`, ...).
  - Added a `weights config.MovieWeights` field to `MovieGoEngine`.
  - `calculateMovieScore()` now uses `e.weights.*` instead of the constants.
  - `classifyMovieStream()`: when `Disable4K` is set, 4K candidates are rejected outright during
    classification (not just given a negative weight). This is necessary because
    `filterMovieStreams()` sorts 4K/1080p candidates into separate pools, and any non-empty 4K
    pool always wins regardless of score — a negative `res_4k` weight alone would NOT have been
    enough to make 1080p win.
  - The TTL recheck heuristic (~line 421) now uses `e.weights.Res4K` instead of the old constant
    as the threshold for detecting whether an existing file is already 4K.

- `internal/syncer/engines/tv_go.go`
  - Same pattern: constants removed, `weights config.TVWeights` field added,
    `calculateQualityScore()` and `classifyStream()` use the weights, including the `Disable4K`
    check and `FullpackBonus` from config.
  - Seeder bonus changed from a stepped scale (+10/+50/+100 at 20/50/100 seeders) to linear
    (`min(seeders, cap) * weight`), the same formula used for Movies.

- `internal/syncer/engines/movies.go`, `internal/syncer/engines/tv.go`
  - Added a `Weights *config.MovieWeights` / `*config.TVWeights` field to `MoviesSyncerConfig` /
    `TVSyncerConfig` and threaded it through to the respective engine constructor.

- `main.go`
  - When building `MoviesSyncerConfig{}` / `TVSyncerConfig{}`, `gc().QualityScoringConfig.Movies`
    / `.TV` is now passed in.

## Known, accepted deviation from previous behavior

With no config changes at all, **Movie** behavior stays 100% identical to the old hardcoded
behavior (all default values copied 1:1 from the old constants, same linear seeder formula).

For **TV**, the defaults change one formula, not just a number: the old seeder bonus was stepped
(`seeders>=100 → +100`, `>=50 → +50`, `>=20 → +10`, otherwise `+0`), the new default is linear
(`min(seeders, 100) * 1`). Both produce the same result at the endpoints (0 and 100+ seeders), but
diverge in between (e.g. at 30 seeders: old `+10`, new `+30`). This could theoretically lead to
slightly different upgrade decisions than before for existing registry entries
(`TVEpisodeEntry.QualityScore`, stored using the old formula), since new candidates are scored
with the new formula.

This deviation was knowingly accepted (not reproduced 1:1 on request) — if it turns out to be an
issue later, the old stepped formula could be restored as a hard fallback, but that would require
additional config fields (thresholds, not just weight/cap) that don't currently exist.

## What was NOT touched

- The watchlist engine (`watchlist_go.go`) and `internal/syncer/quality/scorer.go`
  (`DefaultMovieProfile()`) — out of scope per the original request.
- Pure size/time sanity bounds like `mMovie4KMinGB`, `mMovie4KMaxGB`, `mMovieUpgradePct`, etc.
  These are plausibility filters, not quality weights — let me know if those should become
  configurable too.

## Config example

See `quality_scoring.example.json` — can be dropped in as-is under the top-level `quality_scoring`
key in your existing `config.json`. Both subsections (`movies`, `tv`) are optional; if either is
missing entirely, the engine's built-in defaults apply. Within a section, you can also specify
just a subset of fields — the rest fall back to their defaults.

## Known limitation of this session

Full `go build ./...` with complete module resolution wasn't possible in this sandbox (no network
access to proxy.golang.org). What was verified instead:
- `internal/config/config.go` was built in isolation with stubbed dependencies and compiled
  cleanly.
- All changed files in `engines` were checked with `gofmt` for syntax errors (none found).
- Manually searched for orphaned references to the removed constants (none found).

**Please still run `go build ./...` and any existing tests locally before deploying.**
