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
| **Handles** | system total, busiest process, top 5 as `name (PID)=count`, and **how far the floor rose in 12h** — the number an alert is useless without |
| **Kernel pool** | nonpaged and paged, in bytes and as a **share of physical memory** |
| **Windows events** | 4227 / 4231 — the system stating outright that ports ran out |
| **Reference** | a `PID=name` map sorted by PID, to decode socket alerts |

Plus three graphs: sockets by state, handles and pool, and port headroom against the discovered range.

The dynamic port range is the one quantity that is **measured rather than declared**. A discovery
rule carries its start and size straight into the headroom item and its triggers, so there is
nothing to keep in sync and nothing that can diverge: after `netsh int ipv4 set dynamicport tcp`
everything recomputes itself. The price is known — Zabbix requires a prototype key to contain a
discovery macro, so retuning the range replaces the item rather than updating it and its history
restarts: the absolute trigger recovers within its own window, the forecast needs six hours again.

## What it alerts on

One principle: **a leak is not a large number, it is the absence of a return to baseline.**
There is no absolute threshold here for how many handles or sockets a process may hold: across 28
mixed Windows servers the busiest process held between 1722 and 487426 handles, a spread of 283x, at
which any default is wrong on most of a fleet and the per-host overrides it forces are worse than no
trigger at all. The detectors carry no floors either: the three clauses below already ignore a process
going from four handles to seven, and a floor that stops evaluation at the first clause on 26 hosts out
of 28 is a threshold wearing another name. Thresholds survive only where the ceiling is physical — kernel pool as a share of
RAM, and headroom in the dynamic port range.

| Trigger | Severity | Logic |
|---|---|---|
| Windows confirms TCP port exhaustion | High | event 4227/4231 — the failure already happened. Confirmation, not early warning |
| Dynamic port range is running out | High | free ports below `{$WINEX.PORTS.FREE.PCT}` % of the discovered range |
| Dynamic port range will run out | Warning | `timeleft` on port headroom < `{$WINEX.PORTS.TIMELEFT}` |
| Bound sockets are not returning to baseline | Warning | the 6h floor clears the 12h floor, the 12h floor clears the day's (both by `{$WINEX.GROWTH.FACTOR}`) **and** the 6h window sits above all of the preceding 6h. The middle clause is what separates a leak from a working day |
| Connections are not being closed | Warning | CLOSE_WAIT above threshold, offending PID in `opdata` |
| Reconnect storm | Warning | TIME_WAIT holds more than `{$WINEX.SOCK.TIMEWAIT.PCT}` % of the **discovered** port range; not a leak, and the trigger says so |
| Handles are not returning to baseline | High | the same rule on the busiest process's handles. The only handle leak detector here |
| Nonpaged pool above the ceiling | High | above `{$WINEX.POOL.NONPAGED.PCT}` % of physical memory |
| Paged pool is not returning to baseline | Warning | the same rule on paged pool bytes |
| Nonpaged pool is not returning to baseline | Warning | the same rule on nonpaged pool bytes |
| Paged pool above the ceiling | High | above `{$WINEX.POOL.PAGED.PCT}` % of physical memory |
| Socket probe: no data | Average | **dependency root**: agent down, probe unsupported, or payload truncated |
| Process probe: no data | Average | only the process query broke |
| Kernel pool: no data | Average | either pool counter went missing |
| Kernel pool ratio: no data | Average | the pool shares cannot be computed — counter, memory reading or calculation |
| Port range probe: no data | Average | the dynamic port range stopped being measured |

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
3. There is nothing to calibrate and nowhere to calibrate it: the leak detectors compare a host with
   itself. They reach full sensitivity a day after linking, when the longest window fills, and catch a
   steep climb sooner than that. Before that a detector is **OK**,
   not Unknown: the clause order is chosen so that Zabbix stops evaluating at the first false term and
   never reaches a shifted window or a calculated item that does not exist yet. That guarantee starts
   at the first stored value; between linking and that value every trigger in every template is
   Unknown, and this one is not special.

## Macros

| Macro | Default | Meaning |
|---|---|---|
| `{$WINEX.GROWTH.FACTOR}` | `1.05` | How far a floor must rise - the 6h floor against the 12h, and the 12h against the day. The only number in all four detectors |
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

- **a leak spread across a hundred processes** that never moves the busiest one. The reasoning is
  that such a leak still raises the paged pool, whose ceiling is physical;
- **only handles carry a growth figure in the alert** - `winex.handles.growth`. Sockets and pool
  do not;
- **`Reconnect storm` works only where the dynamic port range was discovered**, since its threshold
  is a share of `{#PORT_NUM}`. If the `MSFT_NetTCPSetting` query never answers, a separate
  `Port range probe: no data` trigger says so and the prototypes depend on it;
- **a genuinely growing workload is indistinguishable from a leak.** The rule sees a floor that does
  not come back down; a server that is honestly getting busier looks the same. That is the price of
  having no thresholds.

Adjacent subjects deliberately left out:

- **restart auditing** - System log (41 / 1074 / 6008 / 6009), presence and age of `MEMORY.DMP`,
  the pending-reboot flag;
- **RDS sessions** - `\Terminal Services\*` counters.

## License

MIT. See [LICENSE](LICENSE).
