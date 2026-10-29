# rst2date

Turns IFS-FESOM restarts stored by chunk number into a dated archive, without touching the
original run.

Runs made with the `DE_CY48R1.0_climateDT_20260326` bundle keep their restarts as

```
<hpcroot>/<expid>/restarts/<SDATE>/<member>/1/  2/  3/  current -> 3
```

where `N/` holds the state at the start of chunk N. `storylines4extremes` and newer RAPS want
one directory per restart date instead:

```
<RESTART_ROOT>/<scenario>/<member>/<YYYYMMDD>/
```

`rst2date` builds that. It reads each chunk's date from the restart itself (rcf `CTIME` plus the
experiment start in the BLS/LAW file names), checks it against `fesom.clock`, and checks the
files the model needs before it starts (the same checks RAPS 1.1.18 makes).

## Install

One file, Python 3, no other dependencies. Copy it anywhere, e.g. to MN5:

```bash
scp rst2date <mn5-login>:<your-scratch>/
```

Run it as `./rst2date ...` from that dir, or by full path.

## The archive it builds

```
<archive>/
├── by-expid/                       the store: one tree per source run, never mixed
│   └── <expid>/<member>/<YYYYMMDD>/   rcf, waminfo, LAW/BLS/srf, fesom_raw_restart, fesom.clock
├── storylines-gen2-run1/           a view: RESTART_ROOT for the current runs
│   └── <scenario>/<member>/<YYYYMMDD> -> ../../../by-expid/<expid>/<member>/<YYYYMMDD>
└── storylines-gen2-run2/           a view: RESTART_ROOT for the reruns
```

- **Store** (`by-expid/`): the restarts, per source run, so you always know where one came from.
- **View**: links only, in the layout `storylines4extremes` reads. Point `RESTART_ROOT` at one.
- Each view slot (`<scenario>/<member>/<date>`) comes from exactly one source. A slot that
  already points somewhere else is refused, never overwritten.

## Usage

```bash
./rst2date <src> <archive> [--view NAME --scenario NAME] [options] [--apply]
```

Nothing changes without `--apply`: run it once without, read the table, then add `--apply`.

| `<src>` | Example | Result |
|---|---|---|
| an SDATE dir | `<hpcroot>/<expid>/restarts/20260701` | every member found in it |
| one member dir | `<hpcroot>/<expid>/restarts/20260701/fc0` | that member only |
| an already dated tree | `<root>/storyline_gen2_restarts/cont` | linked into the view as it is |

| Option | Meaning |
|---|---|
| `--view NAME` | the view to link into, e.g. `storylines-gen2-run1` |
| `--scenario NAME` | the scenario dir in the view: `hist`, `cont` or `tplus2k` (the names `storylines4extremes` reads) |
| `--expid ID` | the source expid, if it cannot be read from `<src>` (it is the dir before `restarts/`) |
| `--members fc0,fc4` | only these members |
| `--chunks 2-4,7` | only these chunk numbers |
| `--mode symlink` | default: links to the original files, no extra space |
| `--mode link` | hard links: no extra space, but only for your own files on the same fileset |
| `--mode copy` | real copies (~120 GB per date at TCo1279); survives the source being deleted |
| `--format date` | default: dirs named `YYYYMMDD`, what `storylines4extremes` reads |
| `--format raps` | dirs named `YYYYMMDD_HHMMSSZ`, what RAPS 1.1.18 reads |
| `--apply` | actually create; without it, a dry run |

`./rst2date -h` prints the same.

## Examples

```bash
A=<scratch>/restart_archive

# a current run, all members, into run1 (dry run, then for real)
./rst2date <hpcroot>/<expid>/restarts/<SDATE> $A --view storylines-gen2-run1 --scenario hist
./rst2date <hpcroot>/<expid>/restarts/<SDATE> $A --view storylines-gen2-run1 --scenario hist --apply

# an existing dated archive, linked as it is
./rst2date <root>/storyline_gen2_restarts/cont $A --view storylines-gen2-run1 --scenario cont --apply

# a rerun, into run2
./rst2date <hpcroot>/<rerun-expid>/restarts/<SDATE> $A --view storylines-gen2-run2 --scenario hist --apply

# only the store, no view
./rst2date <hpcroot>/<expid>/restarts/<SDATE> $A --apply
```

Run it again whenever a run has new chunks: what exists is left alone, only new dates are added.

## Reading the output

```
member source  restart date         tasks  target    action
fc0    2       2026-08-01 00:00:00  1652   20260801  would create in store, would link in view
```

| Action | Meaning |
|---|---|
| `would create in store` / `in store` | will be built in `by-expid/` / is already there |
| `would link in view` / `linked in view` | will be linked / is already linked |
| `no rcf ... left alone` | an empty chunk (a cold start's chunk 1); normal |
| `SKIP: <reason>` | not usable as it is (e.g. dates disagree, files missing) or the view slot is taken |
| `not carried: ...` | RAPS's own copies of the *next* restart, kept in the original only |

Exit code: `0` all fine, `1` some rows were skipped, `2` wrong usage.

## With storylines4extremes

In its `config.sh`, set

```bash
RESTART_ROOT="<scratch>/restart_archive/storylines-gen2-run1"     # or ...-run2
```

and check with its dry run, which creates and submits nothing:

```bash
./storylines4extremes -n -s hist -m all 20260801 20260811 test
```

The event start must be a date that exists in the view (the first of a month).

## Things to know

- **The original run is never changed.** The archive must be outside it; `rst2date` refuses
  otherwise.
- **Symlinks live as long as the source does.** If the run's restarts are cleaned up later, its
  links break. Use `--mode copy` for dates you must keep.
- **A run that is still going:** its newest chunk dir (the one `current` points to) can change
  while the job runs. Leave it out (`--chunks 1-3` while `current -> 4`) or use `--mode copy`.
- **Runs that continue each other** share one date: the last restart of the first run is the
  first restart of the next. That view slot stays with the first run; the second run's copy is
  still kept in its own store.
- **Keep the store and the views together.** View links into `by-expid/` are relative, so move
  the whole `<archive>` dir, never a view on its own.
