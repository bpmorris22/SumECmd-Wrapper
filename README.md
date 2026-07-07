# SumECmd-Wrapper

A single-file GUI for **SUM / User Access Logging triage** with Eric Zimmerman's [SumECmd](https://github.com/EricZimmerman/Sum) — runs SumECmd for you and turns the output into an interactive, suspicion-scored view that answers *"which account, from which source IP, accessed this server, over months."* One `.hta`, no install, part of the [DFIR-Artifact-Finder](https://github.com/bpmorris22/DFIR-Artifact-Finder) wrapper family.

SUM (**S**oftware **U**sage **M**etrics / User Access Logging) is a **Windows Server** feature — workstations have no `Sum\` folder. Its ESE databases under `%SystemRoot%\System32\LogFiles\Sum\` record, per client, which authenticated user reached which server role from which IP, day by day, with a retention that typically spans **years**.

## Quick start

1. Put `SumECmd-Wrapper.hta` anywhere and double-click it — use **Update / download SumECmd** to fetch the engine, or point it at an existing copy (KAPE `Modules\bin` is found automatically).
2. Point the input at a collected `...\System32\LogFiles\Sum\` folder (KAPE / Velociraptor trees work), or at a `Current.mdb` inside one — a bare `.mdb` resolves to its parent folder. The folder must contain `SystemIdentity.mdb` (+ `Current.mdb` + any archived-year `{GUID}.mdb`) so role GUIDs resolve to friendly names.
3. Confirm the **Target hostname** guess, then **Process → analyze**. Or **Load existing CSV…** to view a SumECmd CSV you already have.

## Dirty ESE databases — auto-repair on a copy

SUM databases pulled from a **live server are routinely dirty** (ESE dirty shutdown) and SumECmd aborts. The wrapper detects this and offers a **repair on a COPY**: it copies every `*.mdb` (WITHOUT the `.log/.chk/.jrs` transaction files — those are what break replay), runs `esentutl /p <db>.mdb /o` on each, and re-runs SumECmd against the repaired copy. **Your original evidence is never touched.** If a database was written by a *newer* Windows build than the analysis box, the local ESE engine can't open it even after repair — parse it on a Windows build at least as new as the source server (a VM works).

## Views & scoring

- **Clients** (default) — one row per **user × source IP × role**, score-ranked; **Timeline** — the same rows by last seen; **Users** — grouped per authenticated account (distinct IPs, distinct roles, first/last seen, max score). Sortable, filterable, resizable columns, per-day detail pane, CSV export.
- Suspicion scoring (threshold ≥ 3): **IOC/keyword hit** (+3), **external / routable source IP** (+2, not RFC1918 / loopback / link-local), **admin-pattern account** (+1: `admin`, `adm-`, `da-`, `dadmin`, `svc`, `-adm`), **single-day burst** (+1, first-seen day = last-seen day), **routable IPv6** (+1, uncommon in these environments).

## Command line

```
mshta "SumECmd-Wrapper.hta" "<inputOrCsv>" ["<outDir>"] [/auto]
```

- `<input>` — a `.csv` (auto-loads into the viewer), a `Sum\` folder, or a `.mdb` inside one (prefilled; processed with `/auto`).
- `<outDir>` — CSV output directory (optional; defaults to `_Processed\<host>\SumECmd` next to the app).
- **Target hostname** is required before processing — it names the `_Processed\<host>\SumECmd` output folder next to the app (family convention shared with the DFIR-Artifact-Finder, so processed evidence is visible per host per tool). Guessed from `Collection-<host>-…` paths or a passed `_Processed\<host>\` outDir — overwrite the guess if it's wrong.
- **Shared IOC list** — an `IOC.txt` next to the app (one term per line, `#` comments) is auto-merged into the IOC box at launch; one list covers the whole toolkit and terms you paste locally are kept.
- **Run provenance + triage summary** — every successful run appends a `runinfo.json` entry (app, host, input path, files) in the output folder, including a triage summary (entries, flagged count, max score, top hits); the DFIR-Artifact-Finder shows these per host in its inventory, even for standalone runs.

## Notes

- SumECmd needs the whole `Sum\` folder together — `SystemIdentity.mdb` maps role GUIDs to names, `Current.mdb` holds the live year, and each archived `{GUID}.mdb` is a prior year chained in.
- All timestamps are UTC. SUM **retention typically spans years** — mind the date columns (and the from/to Last-seen filter) when scoping a window.
- Columns are resizable — drag a header's right edge; double-click the edge to reset. Widths are remembered per view.
- Requires Windows (mshta / IE-JScript host + `esentutl`).

MIT © 2026 Ben Morris
