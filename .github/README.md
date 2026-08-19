# Quality-Weights konfigurierbar machen (Movie + TV Sync)

## Was geändert wurde

- `internal/config/config.go`
  - `QualityWeights`/`TVQualityWeights` (unbenutzt) ersetzt durch `MovieWeights` und `TVWeights`,
    die 1:1 zu den tatsächlichen Scoring-Feldern der Movie-/TV-Engines passen.
  - Beide haben eine eigene `UnmarshalJSON`, die zuerst die Defaults (`DefaultMovieWeights()` /
    `DefaultTVWeights()`) setzt und dann nur die in der JSON vorhandenen Felder überschreibt.
    Ein Feld weglassen == Default behalten.
  - Neues Feld `disable_4k` (bool) in beiden Structs.
  - Neue Felder `seeder_weight` / `seeder_cap` (int) in beiden Structs: Beitrag zum Score ist
    `min(seeders, seeder_cap) * seeder_weight`.

- `internal/syncer/engines/movie_go.go`
  - Konstanten-Block (`mMovie4KBase`, `mMovieHDRBonus`, ...) entfernt.
  - Neues Feld `weights config.MovieWeights` auf `MovieGoEngine`.
  - `calculateMovieScore()` nutzt `e.weights.*` statt Konstanten.
  - `classifyMovieStream()`: wenn `Disable4K` gesetzt ist, werden 4K-Kandidaten schon bei der
    Klassifizierung verworfen (nicht nur negativ gewichtet!) — notwendig, weil `filterMovieStreams()`
    4K/1080p in getrennte Pools sortiert und ein nicht-leerer 4K-Pool immer gewinnt, unabhängig vom
    Score. Eine negative `res_4k`-Gewichtung allein hätte NICHT gereicht, um 1080p gewinnen zu lassen.
  - TTL-Recheck-Heuristik (Zeile ~421) nutzt jetzt `e.weights.Res4K` statt der alten Konstante als
    Schwellenwert, um zu erkennen ob eine vorhandene Datei bereits 4K ist.

- `internal/syncer/engines/tv_go.go`
  - Gleiches Muster: Konstanten raus, `weights config.TVWeights`-Feld, `calculateQualityScore()` und
    `classifyStream()` nutzen die Weights, inkl. `Disable4K`-Check und `FullpackBonus` aus Config.
  - Seeder-Bonus von gestuft (+10/+50/+100 bei 20/50/100 Seedern) auf linear
    (`min(seeders, cap) * weight`) umgestellt, gleiche Formel wie bei Movies.

- `internal/syncer/engines/movies.go`, `internal/syncer/engines/tv.go`
  - `Weights *config.MovieWeights` / `*config.TVWeights` Feld in `MoviesSyncerConfig` /
    `TVSyncerConfig` ergänzt und bis zum jeweiligen Engine-Constructor durchgereicht.

- `main.go`
  - Beim Bauen von `MoviesSyncerConfig{}` / `TVSyncerConfig{}` wird jetzt
    `gc().QualityScoringConfig.Movies` / `.TV` übergeben.

## Was NICHT angefasst wurde

- Die Watchlist-Engine (`watchlist_go.go`) und `internal/syncer/quality/scorer.go`
  (`DefaultMovieProfile()`) — laut Absprache irrelevant für diesen Use-Case.
- Reine Größen-/Zeit-Sanity-Grenzen wie `mMovie4KMinGB`, `mMovie4KMaxGB`, `mMovieUpgradePct` etc.
  Das sind Plausibilitätsfilter, keine Qualitätsgewichtungen — falls die auch konfigurierbar werden
  sollen, sag Bescheid.

## Config-Beispiel

Siehe `quality_scoring.example.json` — kann 1:1 unter dem Top-Level-Key `quality_scoring` in deine
bestehende `config.json` übernommen werden. Beide Untersektionen (`movies`, `tv`) sind optional;
fehlt eine ganz, gelten die Engine-internen Defaults. Innerhalb einer Sektion kannst du auch nur
einzelne Felder angeben, der Rest bleibt beim Default.

## Bekannte Einschränkung dieser Session

In dieser Sandbox war kein `go build ./...` mit vollständiger Modul-Auflösung möglich (kein
Netzwerkzugriff auf proxy.golang.org). Ich habe:
- `internal/config/config.go` isoliert mit Stub-Dependencies erfolgreich gebaut (kompiliert sauber).
- Alle geänderten `engines`-Dateien mit `gofmt` auf Syntaxfehler geprüft (keine gefunden).
- Manuell nach verwaisten Referenzen auf die entfernten Konstanten gesucht (keine gefunden).

**Bitte vor dem Deploy trotzdem `go build ./...` und vorhandene Tests lokal laufen lassen.**
