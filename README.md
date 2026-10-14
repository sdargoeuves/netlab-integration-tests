# parallel-test

Run the netlab integration tests (`netlab/tests/integration/...`) for one device type, several
tests at a time. Each running test gets its own lab instance through netlab's **multilab** plugin,
so the labs don't clash.

Each test goes through the same steps as netlab's `tests/integration/device-module-test`:
`create` → `up` → `initial` (configuration) → `validate` → `down --cleanup`.
Results are kept in `logs/`: an overview across all runs, a summary per run, and a log per step.

- [Quick start](#quick-start)
- [Testing the whole test suite](#testing-the-whole-test-suite)
- [Settings files: one per device](#settings-files-one-per-device)
- [Reading the results](#reading-the-results)
- [Troubleshooting](#troubleshooting)
- [Stopping a run, leftover labs](#stopping-a-run-leftover-labs)
- [Options](#options)
- [How it works](#how-it-works)

## Quick start

Requirements:

- Python 3.9 or later with PyYAML. The Python environment you installed netlab in has it.
- netlab with containerlab. Your user must be in the `clab_admins` group, so containerlab runs
  without sudo.
- For the whole test suite (`--all`): a netlab git clone (`git clone https://github.com/ipspace/netlab`),
  because the integration tests aren't installed with netlab.

First-time setup: create your per-host settings from the example, and set your netlab git clone in it
if you have one (otherwise the `netlab` on your `PATH` is used):

```bash
cd test-netlab
cp settings/local.yml.example settings/local.yml
$EDITOR settings/local.yml        # netlab: ~/src/netlab, plus any replacement images
```

Everything device-specific lives in a settings file, so a run only needs that file and the tests:

```bash
cd ~/code/quick-netlab-lab/test-netlab

# One test
./parallel-test --settings settings/asav.yml ../netlab/tests/integration/bgp/01-ebgp-session.yml

# A whole directory (every [0-9]*.yml in it), 5 labs at a time
./parallel-test --settings settings/asav.yml -j 5 ../netlab/tests/integration/bgp

# Several directories and files mixed
./parallel-test --settings settings/asav.yml -j 5 \
  ../netlab/tests/integration/routing ../netlab/tests/integration/ospf/ospfv2 \
  ../netlab/tests/integration/initial/04-mtu.yml
```

At the end you get a summary table, also saved in `logs/latest/summary.txt`.

## Testing the whole test suite

One command runs every test of the suite for a device:

```bash
./parallel-test --settings settings/asav.yml --all -j 5 --name asav-9.18-full
```

`--all` selects every directory of the test suite: 262 tests in 28 directories, including
`ospf/ospfv2` and `ospf/ospfv3`. Most of them can't run on a given device, because it doesn't
support a feature the test needs (VRFs, routing policies, OSPFv3...). netlab rejects those at the
`create` step within about a second. They're reported as `unsupported`, not as failures, and their
slot moves on to the next test straight away. Only the supported tests start a lab: for ASAv 9.18,
28 of the 262.

Choose `-j` for your host: VM-based devices like ASAv need a lot of CPU while booting. With 5 ASAv
at a time, each one took about 5 minutes to become healthy, and the ASAv tests that passed took
4–10 minutes each (about 7 on average). 28 tests at `-j 5` is roughly 40 minutes. Start with
`-j 3`–`5`.

Run it in tmux, and detach (Ctrl-b d) rather than closing the pane: the run keeps going. Closing
the pane stops the run (cleanly, tearing down its labs).

### Review the results

```bash
cat logs/latest/summary.txt                                    # tests that ran; failures with their log
cat logs/latest/unsupported.txt                                # what the device can't run, and why
awk -F'\t' '$8=="fail"' logs/history.tsv | column -t -s$'\t'   # real failures across all runs
```

To debug a single failure, see [Troubleshooting](#troubleshooting).

### Rerun the failures

```bash
./parallel-test --settings settings/asav.yml --from-run latest --only fail -j 5 --name asav-rerun
```

`--from-run` takes the tests of a previous run: a run name (a folder in `logs/`), `latest`, or a
path to a run folder. `--only` selects them by status, as a comma-separated list of `ok`,
`warning`, `unsupported`, `fail` and `aborted`. Without `--only`, every test except `unsupported`
is selected. You can combine `--from-run` and `--all` with explicit test files and directories; a
test selected twice runs once.

### Optional: see what's supported before a long run

To get the list of supported tests without starting any lab, run only the `create` step. It takes
a few minutes for the whole suite:

```bash
./parallel-test --settings settings/asav.yml --all --steps create -j 10 --name asav-check
```

`logs/asav-check/summary.txt` lists the tests the device can run, and `unsupported.txt` the others
with the reason. To then run exactly those tests:

```bash
./parallel-test --settings settings/asav.yml --from-run asav-check --only ok -j 5
```

## Settings files: one per device

A settings file holds everything you'd otherwise pass on the command line for a device: device
type, image, extra plugins, and topology values. Example, `settings/asav.yml`:

```yaml
plugin: [ files ]
set:
  defaults.device: asav
  defaults.devices.asav.clab.image: vrnetlab/cisco_asav:9-18-1
  # Longer SSH ready check (retries x delay seconds), useful with several ASAv booting in parallel
  defaults.devices.asav.clab.group_vars.netlab_check_retries: 60
  defaults.devices.asav.clab.group_vars.netlab_check_delay: 10
  # ASAv drops traffic between interfaces with the same security level by default
  nodes.dut*.config.inline: |
    same-security-traffic permit inter-interface
```

To test another device, copy it to `settings/<device>.yml` and adapt it. For example, a cEOS file
only needs `defaults.device: eos` and `defaults.devices.eos.clab.image: ceos:4.35.6M`.

### What you can put in it

- **`plugin`**: netlab plugins added to every test (`--plugin`). `config.inline` needs `files`.
  A plugin the test already uses does no harm.
- **`set`**: dotted topology keys, passed to `netlab create` as `-s key=value`.
  - **`defaults.device`** sets the device under test. `-d` on the command line overrides it.
  - **`defaults.devices.<device>.<provider>.image`** sets the device's image. `-i` overrides it.
  - Values must be a string, number or true/false; lists and dictionaries can't be set this way.
    Multi-line strings are fine.
  - Neither keys nor values may contain `=`, because netlab's `-s` splits on every `=`. The script
    stops with an error if one does.
- **Node wildcards**: the node name in `nodes.<name>...` may be a wildcard. `nodes.dut*...`
  applies to every matching node in each test: `dut`, `dut_a`, `dut_spine`, `dut2`... Several
  tests have more than one device under test.
  - Tests without a matching node skip the setting. Otherwise netlab would add a new node with
    that name to the lab.
  - If a test already defines the value (e.g. its own `config.inline` on `dut`), it is replaced,
    not merged.

  Both cases are noted on screen and in `results.yaml` (`settings_notes`).

### settings/local.yml: this host's settings, included automatically

Device files like `asav.yml` can be shared. `settings/local.yml` holds what is specific to one
machine, and the script always includes it when it exists (`--no-default-settings` leaves it out):

```yaml
# netlab to run the tests with: a netlab git clone on any branch (dev, or a branch you're
# working on; needed for --all), or a netlab executable
netlab: ~/code/quick-netlab-lab/netlab

set:
  # Replacements for netlab default images that aren't available on this host
  defaults.devices.eos.clab.image: ceos:4.34.5M
  defaults.devices.iol.clab.image: vrnetlab/cisco_iol:17.18.02
```

- **`netlab`**: which netlab runs the tests: the directory of a netlab git clone, on whatever
  branch you want to test (`dev`, or a branch you're working on), or a `netlab` executable.
  `--netlab` on the command line overrides it. Without either, the `netlab` on your `PATH` is
  used. A relative path is relative to the settings file. The `netlab` key can go in any settings
  file, but it belongs in `local.yml`.
- **Replacement images**: some tests use cEOS or IOL as helper nodes, whatever the device under
  test. If you don't have netlab's default version, map it to one you have here.

`local.yml` is listed in `.gitignore`. `settings/local.yml.example` is the template for others.

The netlab that was used is printed at startup
(`netlab: /home/sa/code/quick-netlab-lab/netlab/netlab (asav-route-import@25a57de6bc, from local.yml)`),
and recorded in `run-info.yaml` (`netlab_path`, `netlab_source`) and in the `netlab` column of
`history.tsv`: the git branch@commit for a git clone, the release (e.g. `26.08`) for an installed
netlab.

Two tests (`ospf/ospfv2/40-area-parameters`, `ospf/ospfv3/40-area-parameters`) set
`quay.io/frrouting/frr:10.2.2` themselves. It's a public image: `docker pull` it rather than
overriding it.

### Which value wins

Values are applied in this order, and the last one wins:

1. `settings/local.yml` (unless `--no-default-settings`)
2. `--settings` files, in the order given
3. `-s` values from the command line
4. `-d` and `-i`

`-i` sets the default image for the whole device type, so helper nodes of the same type (e.g. eos
helper nodes when testing eos) also run it.

The image that actually wins is the one recorded for the run, together with where it came from
(`-i`, `-s`, a settings file name, or `netlab default`). It's printed at startup
(`Image: vrnetlab/cisco_asav:9-18-1 (from asav.yml)`), and recorded in the summary header,
`run-info.yaml` and `history.tsv`. Each run folder also keeps a copy of every settings file used
(`settings-<file>.yml`).

## Reading the results

Everything is kept in `logs/`, at three levels:

```text
logs/
├── history.tsv                     # 1. all runs: one line per test per run
├── latest -> asav-20261013-101500  #    the most recent run
└── asav-20261013-101500/           # 2. one run (or the --name you gave it)
    ├── summary.txt                 #    the tests that ran (printed at the end), failures with their log
    ├── unsupported.txt             #    the tests the device can't run, with the reason
    ├── unsupported.log             #    their full create.log, one after the other
    ├── results.yaml                #    every test, for scripts
    ├── run.log                     #    everything printed on screen
    ├── run-info.yaml               #    command, device, image, netlab branch@commit, settings, test list
    ├── settings-asav.yml           #    copies of the settings files used
    └── bgp-01-ebgp-session/        # 3. one test that ran: one log per step
        ├── create.log
        ├── up.log
        ├── initial.log
        ├── validate.log
        └── down.log
```

Only the tests that ran get a folder. An unsupported test only has a `create.log`, so it's
appended to the run's `unsupported.log` (under a `===== <test> (lab <id>) =====` header) and its
folder is removed. A full `--all` run therefore keeps a few dozen test folders, not 262.

The logs are small text files, so keep them all. Deleting a run folder doesn't remove its lines
from `history.tsv`.

### Test status

| Status | Meaning |
| --- | --- |
| `ok` | All steps passed |
| `warning` | Validation passed with warnings |
| `unsupported` | netlab won't build the test for this device. The `reason` is the first netlab error line, e.g. `does not support module vrf` |
| `fail` | A step failed: `failed_step` says which one (`create`, `up`, `config`, `validate`) |
| `aborted` | Stopped by Ctrl-C, `kill` or a closed terminal. Tests that never started have the reason `not started` |

The script exits with code 1 if any test is `fail` or `aborted`.

### 1. All runs: history.tsv

One tab-separated line per test per run. The columns are `date`, `run`, `device`, `provider`,
`image`, `netlab` (branch@commit), `test`, `status`, `failed_step`, `duration`, `logs` (log folder
relative to `logs/`, empty for unsupported tests), `reason` and `settings`. Because it records the
image and netlab commit, it shows whether a test passes on one image or branch and fails on another.

The file is chronological: each run appends its lines at the end. Within a run, lines are sorted by
status (`fail`, `aborted`, `warning`, `ok`, then `unsupported`), then by test name, so each run's
problems come first when you open the file in a spreadsheet or editor. For another order without
changing the file, use `sort`, e.g. by status then test across all runs:
`sort -t$'\t' -k8,8 -k7,7 history.tsv`.

```bash
cd ~/code/quick-netlab-lab/test-netlab/logs
column -t -s$'\t' history.tsv | less -S                                   # everything
grep 11-ipv6-ebgp history.tsv | column -t -s$'\t'                         # one test over time
awk -F'\t' '$8=="fail"' history.tsv                                       # real failures
awk -F'\t' '$8=="unsupported" {print $7"\t"$12}' history.tsv | sort -u    # what the device can't do
awk -F'\t' '$5 ~ /9-18-1/' history.tsv                                    # one image
```

### 2. One run: summary.txt and unsupported.txt

`summary.txt` (also printed at the end of the run) only lists the tests that actually ran, so the
results aren't lost among the unsupported tests. Failed tests are listed again at the end with the
log to read:

```text
Run asav-20261013-101500  device=asav  image=vrnetlab/cisco_asav:9-18-1 (from asav.yml)  netlab=asav-route-import@25a57de6bc  settings=local.yml,asav.yml

test                                       id status        time  steps
bgp/01-ebgp-session.yml                    13 ok            512s  create=True up=True config=True validate=True cleanup=True
bgp/11-ipv6-ebgp.yml                       11 fail          478s  create=True up=True config=True validate=False cleanup=True

ok: 1  unsupported: 1  fail: 1

Failed (log to read):
  bgp/11-ipv6-ebgp.yml                   validate  .../logs/asav-20261013-101500/bgp-11-ipv6-ebgp/validate.log

1 unsupported test(s), see .../logs/asav-20261013-101500/unsupported.txt
```

`unsupported.txt` lists what the device can't do, sorted by reason. The full `create.log` of each
of these tests is in `unsupported.log`, if you need more than the first error line:

```text
234 test(s) netlab cannot build for asav (full create.log of each: unsupported.log):

routing/30-acl-ipv4.yml                  IncorrectAttr in routing: Device asav (node dut) does not support 'acl' objects (h1_filter)
bgp/09-vrf-localas.yml                   IncorrectValue in modules: Device type asav (node dut) does not support module vrf
...
```

During the run, the screen (and `run.log`) shows a line each time a test starts or finishes a step,
prefixed with its lab ID:

```text
10:15:00 Running 3 test(s), 3 at a time, multilab IDs [11, 12, 13]
10:15:00 Image: vrnetlab/cisco_asav:9-18-1 (from asav.yml)
10:15:02 [11] bgp/11-ipv6-ebgp.yml: start (logs: .../logs/asav-20261013-101500/bgp-11-ipv6-ebgp)
10:15:03 [12] routing/01-locpref.yml: create unsupported: IncorrectAttr in routing: Device asav (node dut) does not support routing polices (policy lp_17)
10:22:40 [11] bgp/11-ipv6-ebgp.yml: validate True
```

A failed step prints `FAILED`. After an interrupt, the steps that were stopped print `interrupted`.

### 3. One test: the step logs

The first line of each log is the command that was run. stdout and stderr are combined.

| Log | Step | Look here for |
| --- | --- | --- |
| `create.log` | `netlab create` | topology errors, unsupported features |
| `up.log` | `netlab up --snapshot --no-config` | containerlab deploy, missing images |
| `initial.log` | `netlab initial` | device readiness (healthcheck, SSH), Ansible configuration errors |
| `validate.log` | `netlab validate` | which validation test failed (errors only, unless `-v`) |
| `down.log` | `netlab down --cleanup` | teardown |

```bash
tail -f logs/latest/bgp-11-ipv6-ebgp/initial.log     # follow a test while it runs
less logs/latest/bgp-11-ipv6-ebgp/validate.log       # why validation failed
```

`results.yaml` has every test (including unsupported ones), one entry per test, with `path`,
`logs` (not for unsupported tests), `failed_step`, `reason`, `settings_notes`, and `labdir` when a
lab folder was kept.

## Troubleshooting

### Debugging one failed test

Rerun it on its own with `--keep-failed`. If it fails again, the lab stays up and its folder is
printed (and recorded as `labdir` in `results.yaml`):

```bash
./parallel-test --settings settings/asav.yml --keep-failed ../netlab/tests/integration/bgp/11-ipv6-ebgp.yml

cd /tmp/test-netlab/<run>/bgp-11-ipv6-ebgp
# use the same netlab as the run (printed at startup, 'netlab:' in local.yml)
netlab connect dut
netlab validate            # rerun validation by hand
netlab down --cleanup      # when you're done
```

### Known problems

| Symptom | Cause and fix |
| --- | --- |
| `initial.log`: `SSH server on node dut ... is not ready after N seconds` | Usually the device is still booting: raise `netlab_check_retries` / `netlab_check_delay` in the settings file (wait = retries × delay). |
| `up.log`: containerlab can't find or pull an image | The image isn't available locally. Add a replacement to `settings/local.yml`, or `docker pull` it if it's public. |
| `up.log`: `failed gleaning v4 and/or v6 addresses from bridge via netlink` and an empty `ERROR` box | Several `containerlab deploy` running at the same moment. Happens only with `--no-lock`: keep the default [provider lock](#provider-lock). |
| `Cannot lock lab status file` or `Cannot read lab status file` | Many labs sharing netlab's status file. Only happens with `--shared-status-file`: run without it (see [Lab status files](#lab-status-files)), or lower `-j`. |
| Labs still running after a run (`docker ps --filter name=clab-ml-`) | A teardown failed or the script was killed. `./parallel-test --cleanup` removes them. |
| Ctrl+C prints `^C` and Enter prints `^M` | The terminal is in raw mode (left by an old version of this script). Type `stty sane` and press Ctrl+J, or close the terminal. |
| Ctrl+C does nothing in VS Code | With text selected, Ctrl+C copies instead of interrupting. Click in the terminal first. |

## Stopping a run, leftover labs

- **Ctrl-C once**: tests that haven't started are cancelled, running steps are interrupted, every
  lab that was started is torn down, and the reports are written. This takes as long as the slowest
  teardown, usually under a minute.
- **Ctrl-C twice**: quit immediately. Teardowns that already started finish in the background, and
  the script lists the labs that might still be up, with the command to remove each one.
- **From another terminal**: `kill <pid>` (`pgrep -f parallel-test`) works like one Ctrl-C. So does
  closing the terminal or tmux pane.

If the script was killed without a chance to clean up (`kill -9`, crash), or a teardown failed,
labs stay up. Remove them with:

```bash
./parallel-test --cleanup
```

This finds labs through Docker, not through a netlab status file: every `clab-ml-<id>-*` container
whose `clab.yml` is under `/tmp/test-netlab`. It first stops any leftover netlab/Ansible process in
the lab folder. It then runs `netlab down --cleanup` when the lab folder still has its netlab
snapshot, or `containerlab destroy --name ml-<id>` when it doesn't. Teardown logs go to the test's
log folder, or to `logs/cleanup/` when that folder is unknown.

Your other netlab labs aren't touched. `--cleanup` refuses to run while another `parallel-test` run
is active, because it would take that run's labs down too.

The test labs don't appear in `netlab status` (see [Lab status files](#lab-status-files)). To see
what's running: `docker ps --filter name=clab-ml-`.

## Options

| Option | Default | Meaning |
| --- | --- | --- |
| `tests...` | | Test files and/or directories (a directory is matched against `--limit`) |
| `--all` | | Every directory of the netlab integration test suite |
| `--from-run` | | Tests of a previous run: run name, `latest` or run folder |
| `--only` | `ok,warning,fail,aborted` | With `--from-run`: statuses to select |
| `--settings` | | Settings file (can be repeated), see [Settings files](#settings-files-one-per-device) |
| `-d, --device` | `defaults.device` from the settings | Device type to test |
| `-i, --image` | from the settings, else netlab default | Image of the device under test (wins over everything) |
| `-s, --set` | | Extra `netlab create -s` value (can be repeated, applied after settings files) |
| `--no-default-settings` | off | Don't include `settings/local.yml` |
| `-j, --jobs` | `3` | Maximum number of labs running at the same time |
| `-l, --limit` | `[0-9]` | Pattern for test file names in directories (`-l 1` = `1*.yml`) |
| `--steps` | `create,up,initial,validate` | Steps to run (`create` = support check, `create,up` = boot check) |
| `--name` | `<device>-<timestamp>` | Run name, used as the run folder name |
| `-p, --provider` | `clab` | Virtualization provider |
| `--keep-failed` | off | Leave failed labs running for troubleshooting |
| `--cleanup` | | Tear down labs left by a run that died, then exit |
| `--ids` | | Lab IDs to use, e.g. `21,22,23` (default: free IDs from `--first-id`) |
| `--first-id` | `11` | First lab ID when `--ids` isn't given |
| `--logdir` | `test-netlab/logs` | Where results are kept |
| `-w, --workdir` | `/tmp/test-netlab` | Where lab files are generated (deleted after teardown) |
| `--netlab` | `netlab:` from the settings, else `netlab` on `PATH` | netlab git clone directory or netlab executable to use (`--netlab-dir` also works) |
| `--lock` / `--no-lock` | `~/.netlab/lab.lock` | Lock that makes `containerlab deploy`/`destroy` take turns |
| `--shared-status-file` | off | Register labs in netlab's shared status file instead of one per lab (see [Lab status files](#lab-status-files)) |
| `--sudo` | off | Ask for the sudo password once and keep it valid (only for tests using `tc` or clab Linux bridges) |
| `-v, --verbose` | off | Full `netlab validate` output |

## How it works

### Lab instances

Each running test gets a multilab ID: `netlab create` is called with
`--plugin multilab -s defaults.multilab.id=N`. The lab is then named `ml-N`, its containers are
`clab-ml-N-<node>`, and its management subnet is `192.168.N.0/24`. When a test is done and its lab
is torn down, the ID is reused for the next test.

- IDs already in use are skipped: IDs with running `clab-ml-<id>-*` containers, and IDs listed in
  netlab's shared status file `~/.netlab/status.yaml` (your other labs). 121 is never used, because
  it would overlap netlab's default `192.168.121.0/24`.
- multilab replaces the lab `name` and `addressing.mgmt`. Tests that set these themselves get the
  multilab values.

### Lab status files

netlab records running labs in a status file, `~/.netlab/status.yaml` by default, and locks it on
every change. With many labs sharing that file, netlab commands fail with
`Cannot lock lab status file` (the lock stayed busy too long) or
`Cannot read lab status file` (the file was caught while being rewritten). At `-j 50`, that made
dozens of tests fail, including teardowns, and the file lost track of labs that were still running.

So each test gets its own status file, `netlab-status.yaml` in its lab folder
(`-s defaults.lab_status_file=...`). Only that lab ever uses it, so the lock is always free and
`-j` is only limited by your host.

The price: the test labs don't appear in `netlab status`. That's why the script finds IDs in use
and leftover labs through Docker instead. `--shared-status-file` goes back to netlab's shared file,
if you want to see the labs in `netlab status` and run only a few at a time.

### Folders

Each test has a log folder, `logs/<run>/<suite>-<test>/`, which is kept. It also has a lab folder,
`/tmp/test-netlab/<run>/<suite>-<test>/`, where netlab generates `clab.yml`, the snapshot,
`node_files/` and so on. The lab folder is deleted after a successful teardown, and kept with
`--keep-failed` or when a teardown fails.

### Test steps

The steps are the same as in `device-module-test`. If a test defines `defaults.netlab.integration`
(its own `up`, `initial` or `validate` command, or steps to `skip`), the script uses that.

Some of those commands are scripts that call plain `netlab` (e.g. `initial/10-reload` runs
`netlab up`, `netlab collect` and `netlab config`). So that they use the same netlab as the run,
not whatever `netlab` comes first on your `PATH`, each run creates a small `netlab` wrapper in
`/tmp/test-netlab/<run>/.bin/` and puts it first on `PATH` for every step. The wrapper is removed
at the end of the run.

A test is `unsupported` when `netlab create` rejects it only with `IncorrectValue`,
`IncorrectAttr` or `IncorrectType` errors: netlab's way of saying the device can't do something.
Any other `create` error (YAML, missing values, crashes) is a `fail`.

### Provider lock

By default, labs are created with `-s defaults.multilab.lock=~/.netlab/lab.lock`, so netlab runs
`containerlab deploy` and `containerlab destroy` through `flock`, one at a time. Each takes only a
few seconds: containerlab returns once the containers and links exist. The devices still boot in
parallel, because waiting for them happens later in `netlab initial`, outside the lock. Without
the lock, around 10 deploys starting in the same second made one of them fail.

### Processes and the terminal

Each netlab step runs in its own session, with an empty pipe as input instead of your terminal:

- Ctrl-C reaches only the script, which decides what to interrupt and tears labs down cleanly.
- No step can change your terminal's settings. The script also restores them when it exits.
- The pipe stays open until the step ends, so programs never see end-of-input. This matters for
  netlab's ASAv SSH ready check: with `/dev/null` as input, ssh fails with code 255 on every
  attempt, and the check gives up even though SSH on the ASAv is working.

### sudo

The script doesn't use sudo unless you pass `--sudo`. With containerlab, netlab only needs sudo for
`tc` (links with traffic control) and some Linux bridge settings (links shared by several nodes).
With `--sudo`, you're asked for the password once at the start instead of several labs prompting
at the same time.
