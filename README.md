# fpl-data

Weekly output of a Dixon-Coles + Monte Carlo Fantasy Premier League model.
Published so a phone app can read it; there is no server and no API.

Fetch `manifest.json` first — it carries the season, gameweek, model version
and a sha256 per file, so a client re-downloads only what changed.

```
manifest.json
latest/      what to read by default
  players.json      every player: per-gameweek xP, outcome buckets, rate metrics
  fixtures.json     team-fixtures: xGF, xGA, P(clean sheet), odds/model source
  teams.json        attack, defence, Elo, home advantage
  draws_gw<NN>.i8   1,000 int8 simulation draws per player
  draws_index.json  row order and shape for the draws file
gw/<NN>/     the same five files, frozen, one folder per gameweek
```

## The draws file

`draws_gw<NN>.i8` is a raw `int8` array, `rows × cols` as given in
`draws_index.json`, row order given by its `ids`. A gameweek score fits in one
signed byte (−3 covers a red card plus an own goal).

Expected points are additive, so a squad's xP is the sum of its players'.
**Probabilities are not** — the model correlates players through a shared
attack shock and the clean sheet, so summing marginal haul probabilities
understates a squad's real spread. Sum the draw rows instead and you have the
correct joint distribution.

```python
import json, numpy as np
idx = json.load(open("latest/draws_index.json"))
a = np.fromfile("latest/draws_gw04.i8", dtype=np.int8).reshape(idx["rows"], idx["cols"])
row = {pid: i for i, pid in enumerate(idx["ids"])}
squad = a[[row[p] for p in my_eleven]].sum(axis=0)
squad.mean(), np.percentile(squad, [10, 90]), (squad >= 60).mean()
```

## What is not here

Anything specific to a manager. Leagues, squads, rivals' picks and live scores
come from the public FPL API on the device. This repo holds only what FPL does
not publish: the model's view.

Not affiliated with the Premier League or FPL.
