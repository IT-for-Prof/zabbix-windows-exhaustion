# Zabbix Windows resource exhaustion

> Universal Zabbix 7.0 template: Windows resource exhaustion and leaks — sockets, handles, kernel pool —
> **attributed per process**.

Answers the question "who on this machine is leaking, and what is running out". Not tied to SQL,
to any application, or to any server role. Stock agent keys only: `wmi.getall`, `wmi.get`,
`perf_counter_en`, `eventlog`. **No agent configuration changes**, no third-party tools.

[Русская версия → README.md](README.md)

**Author:** Konstantin Tyutyunnik · [itforprof.com](https://itforprof.com) · MIT

## Why

The stock Windows templates (`Windows by Zabbix agent` / `... active`) collect **no** socket, handle,
paged-pool or per-process metric at all — only `proc.num[]` and `\System\Threads` as totals. That
leaves three common Windows server failures invisible until the machine is already down:

- **dynamic port range exhaustion** — the server is healthy but applications can no longer connect.
  The process responsible is identified only by the **Bound** socket state, which neither `netstat`
  without `-q` nor `net.tcp.socket.count` reports;
- **a handle leak in one process** — grows over hours, visible only per process;
- **nonpaged pool growth** — the counter is usually collected, but no stock template has a trigger
  on it.

The diagnostic this template automates is the canonical one: Microsoft's TCP port exhaustion guidance
prescribes `netstat -anobq` and finding the process with the most **BOUND** entries. Here that runs
continuously and ahead of time, rather than after the fact from events 4227/4231.

## What it collects

24 items per host plus one discovery rule. Two actual WMI queries every five minutes, masters with
`history: 0` — the raw JSON is never stored, and everything else is derived from a single parse by
dependent items.

| Area | Metrics |
|---|---|
| **Sockets** | total, established, CLOSE_WAIT, TIME_WAIT, **Bound** from `MSFT_NetTCPConnection` |
| **Who holds them** | top 5 processes by Bound and by CLOSE_WAIT, formatted for trigger `opdata` |
| **Dynamic ports** | free headroom. The range is **discovered** from `MSFT_NetTCPSetting`, never declared as a macro |
| **Handles** | total, per-process maximum, top 5 as `name (PID)=count` |
| **Kernel pool** | nonpaged and paged, in bytes and as a **share of physical memory** |
| **Windows events** | 4227 — the system stating outright that ports ran out; 4231 — 4-tuple reuse, which is not that |
| **Reference** | a `PID=name` map sorted by PID, to decode socket alerts |

Plus three graphs: sockets by state, handles and pool, and port headroom against the discovered range.

The dynamic port range is the one quantity that is **measured rather than declared**. A discovery
rule carries its start and size straight into the headroom item and its triggers, so there is
nothing to keep in sync and nothing that can diverge: after `netsh int ipv4 set dynamicport tcp`
everything recomputes itself. The price is known — Zabbix requires a prototype key to contain a
discovery macro, so retuning the range replaces the item rather than updating it and its history
restarts: the absolute trigger recovers within its own window, the forecast needs six hours again.

## What it alerts on

One principle: **an alert must name a limit.** Not "is it climbing" but "how far is this from the
thing it runs out of". The difference is not philosophical: across 28 mixed Windows servers the growth
detectors - there were four of them here - reported nine hosts at once, three of them reopening twice
within two hours, while the nearest resource anywhere on the fleet was thirty-five days from any
limit. Handles occupied 2.5% of their hard limit, ports 1.4% of the range, nonpaged pool 4.9% of RAM.
Growth on a resource with 97% headroom is a graph, not an alert, and no threshold turns it into one.

The growth detectors were therefore removed outright, leaving three kinds of condition:

- **a ceiling** — a share of a physical limit: kernel pool against RAM, port headroom against the
  discovered range, one process's handles against `{$WINEX.HANDLES.CEILING}`;
- **a forecast** — `timeleft` to port exhaustion, the one place here where running out is real and fast;
- **a fact** — Windows itself logged 4227/4231; the failure already happened.

The handle ceiling is the only number here not derived from physics: the per-process limit is
16777216 and nothing reaches it, kernel memory bounds the machine first. The fleet spread is 1722 to
487426 handles for the busiest process, a factor of 283, so one default is not right everywhere.
Where a host legitimately runs a large holder, override `{$WINEX.HANDLES.CEILING}` **on that host**.
That decision is made once, by a person, in daylight, and it is the whole calibration this needs.

| Trigger | Severity | Logic |
|---|---|---|
| Windows confirms the dynamic port range is exhausted | High | event 4227 — no ports left in the range, outbound connections are failing now |
| Local endpoint reused faster than TIME_WAIT drains | Warning | event 4231 — 4-tuple reuse. The range is nearly empty when it fires; the fix is in the application, not a wider range |
| Dynamic port range is running out | High | free ports below `{$WINEX.PORTS.FREE.PCT}` % of the discovered range |
| Dynamic port range will run out | Warning | `timeleft` on port headroom < `{$WINEX.PORTS.TIMELEFT}` |
| Connections are not being closed | Warning | CLOSE_WAIT above threshold, offending PID in `opdata` |
| Reconnect storm | Warning | TIME_WAIT holds more than `{$WINEX.SOCK.TIMEWAIT.PCT}` % of the **discovered** port range; not a leak, and the trigger says so |
| Handles in one process are above the ceiling | Average | one process's 24h **floor** is above `{$WINEX.HANDLES.CEILING}`. The floor, not the last value: a process that peaks at four times its trough every day is not reported for its peaks. Average, not High — a standing high count is something to schedule a restart for, not to wake up for |
| Nonpaged pool above the ceiling | High | above `{$WINEX.POOL.NONPAGED.PCT}` % of physical memory |
| Paged pool above the ceiling | High | above `{$WINEX.POOL.PAGED.PCT}` % of physical memory |
| Socket probe: no data | Warning | **dependency root**: agent down, probe unsupported, or payload truncated |
| Process probe: no data | Warning | only the process query broke |
| Kernel pool: no data | Warning | either pool counter went missing |
| Kernel pool ratio: no data | Warning | the pool shares cannot be computed — counter, memory reading or calculation |
| Port range probe: no data | Warning | the dynamic port range stopped being measured |

Each collection chain — sockets, processes, kernel pool, event log — has **its own dependency root**.
A dependency here means "the parent's failure makes this signal meaningless", not "these tend to fail
together": a stalled WMI query must not silence a pool alert whose data is still arriving. The price is
three events instead of one during a total agent outage.

Triggers that name an offender carry a "PID to name map" link and a `pidmap_item` tag — one click, or
one API call, from the problem to the PID lookup.

## Requirements

- Windows 8 / Server 2012 or newer — earlier releases have no `root\StandardCimv2` with
  `MSFT_NetTCPConnection`.
- Zabbix agent or agent 2, default configuration. Verified on agent 2 7.0.18 / Windows Server 2025.
- `perf_counter_en` only — on a localized Windows the non-English key finds nothing.
- The server or proxy must reach the agent: everything except the event log is a passive check.
  `eventlog[]` is active-only and silently collects nothing where active checks are not set up.

## Installation

1. Import `template/template_windows_resource_exhaustion.yaml`
   (`Data collection → Templates → Import`, `Create new` + `Update existing`, `Delete missing` off).
2. Link it to Windows hosts **in addition** to the stock template, not instead of it.
3. Walk the hosts where `Handles in one process are above the ceiling` fired and decide once, per
   host: legitimately large holder, or finding. If legitimate, override `{$WINEX.HANDLES.CEILING}` on
   the host. That is the only threshold that may ever need an override: ports and pool are shares of
   what the host reports about itself — the discovered port range and measured memory — while a handle
   count has no denominator anywhere in Windows. The norm belongs to the process name, not the host:
   `dns.exe` holds 10 400 on every controller, a healthy `lsass.exe` holds 2 000–2 500, `searchd`
   legitimately holds 40 000–58 000. No formula covers all of them: a share of the 16 777 216
   per-process cap sits four orders of magnitude above anything observed, and comparing a host with
   itself fires on a fifty-handle move.
   The handle ceiling reaches full sensitivity a day after linking, when the 24-hour window fills.
   Before that the trigger is silent and stays **OK**, not Unknown: the clause order is chosen so that
   Zabbix stops evaluating at the first false term. That guarantee starts at the first stored value;
   between linking and that value every trigger in every template is Unknown, and this one is not
   special.

## How alerts close

The way Zabbix closes anything by default: **when the expression stops holding**. No trigger carries a
`recovery_expression`, and that is a decision rather than an omission.

A separate recovery expression is a second, independent condition. Wherever it can be true at the same
instant as the firing condition, Zabbix opens and closes the problem in a loop. Measured on the fleet's
own history, at the cadence each metric is actually collected at, that cost the growth detectors 57
problems against 19. Those detectors are gone, but the rule still binds anything added here later.

Ceiling triggers close when the quantity drops back below the ceiling: pool below its share of RAM, a
process's 24-hour floor below `{$WINEX.HANDLES.CEILING}`. A process that gets restarted releases its
alert within a day - exactly the time the new, lower value needs to push the old one out of the
24-hour window. `manual_close` is on where closing by hand makes sense.

Both event-log triggers close themselves 30 minutes after the last log entry: they report a fact,
not a state.

## Macros

| Macro | Default | Meaning |
|---|---|---|
| `{$WINEX.HANDLES.CEILING}` | `100000` | Handles one process may hold before it is reported regardless of trend. Read off the 24h floor rather than the last value, so a process that peaks high and falls back every day is not reported for its peaks. Measured: the largest floor on a healthy host was a search indexer at 58000 |
| `{$WINEX.NODATA.PERIOD}` | `15m` | Silence tolerated on a probe. Keep at ≥ 3 × `{$WINEX.PROBE.INTERVAL}` |
| `{$WINEX.POOL.NONPAGED.PCT}` | `20` | Nonpaged pool ceiling, percent of physical memory |
| `{$WINEX.POOL.PAGED.PCT}` | `15` | Paged pool ceiling, percent of physical memory |
| `{$WINEX.PORTS.FREE.PCT}` | `12` | Minimum acceptable headroom, as a percent of the discovered range |
| `{$WINEX.PORTS.TIMELEFT}` | `1d` | How far ahead the exhaustion forecast looks |
| `{$WINEX.PROBE.INTERVAL}` | `5m` | WMI polling interval |
| `{$WINEX.SOCK.BOUND.FLOOR}` | `500` | Bound sockets a host must already hold before the port forecast may speak. The forecast only: the leak detectors carry no floor |
| `{$WINEX.SOCK.CLOSEWAIT.MAX}` | `200` | CLOSE_WAIT sockets host-wide |
| `{$WINEX.SOCK.TIMEWAIT.PCT}` | `25` | TIME_WAIT as a percent of the discovered port range, which is what they compete for |
| `{$WINEX.WMI.TIMEOUT}` | `30s` | WMI item timeout. The Zabbix ceiling for an agent item; the cost of enumerating connections scales with their number, so there is nothing above this |

Pool ceilings are a share of memory rather than a byte count, because on x64 it is memory that bounds
the nonpaged pool. A fixed ceiling is wrong in both directions at once: tight on a large host that
legitimately runs a large pool, and so loose on a very large one that a real runaway never reaches it.

There is deliberately no macro for the port range itself — it is discovered. It is set with
`netsh int ipv4 set dynamicport tcp start=N num=M`, not through the registry: `MaxUserPort` has been
ignored since Vista. The threshold is a share rather than a count, because widening the range makes
any fixed count mean something different.

## Scope

What this template deliberately does **not** catch:

- **a leak before it reaches the ceiling.** This is the main and deliberate loss. A process growing
  1000 handles a day from a base of 12000 goes unreported for 73 days, until it crosses
  `{$WINEX.HANDLES.CEILING}`. The growth detector that caught this was here and was removed: on a live
  fleet it reported nine hosts out of 28 at once and reopened on three. If a particular host is worth
  watching sooner, the answer is to lower the ceiling **on that host**, not to bring the general rule
  back;
- **there is no forecast for handles or pool.** `timeleft` only works where history runs deeper than
  the forecast horizon; on three days of data two ways of computing it disagree by a factor of 20. The
  only forecast here is on ports, where the horizon is a day. Revisit once two or three weeks have
  accumulated: Zabbix 7.0 has `trendavg`, `timeleft`, `baselinewma` and `baselinedev`, all native;
- **a leak spread across a hundred processes** that never moves the busiest one. The reasoning is
  that such a leak still raises the paged pool, whose ceiling is physical;
- **`Reconnect storm` works only where the dynamic port range was discovered**, since its threshold
  is a share of `{#PORT_NUM}`. If the `MSFT_NetTCPSetting` query never answers, a separate
  `Port range probe: no data` trigger says so and the prototypes depend on it;
- **a genuinely growing workload is indistinguishable from a leak** - which is exactly why the ceiling
  is set per host rather than derived from a trend: "too much" is decided by a person who knows what
  the server is for;
- **GDI and USER objects are not collected.** Their per-process limit is a hard 10000, and on terminal
  servers that is a likelier cause of failure than kernel handles. A known gap.

Adjacent subjects deliberately left out:

- **restart auditing** - System log (41 / 1074 / 6008 / 6009), presence and age of `MEMORY.DMP`,
  the pending-reboot flag;
- **RDS sessions** - `\Terminal Services\*` counters.

## License

MIT. See [LICENSE](LICENSE).
