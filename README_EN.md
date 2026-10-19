# Zabbix Windows resource exhaustion

> Zabbix 7.0 template: resource exhaustion on Windows - sockets, handles, kernel pool - attributed per process.

The template shows which process on a machine is using up what. It is not tied to a product or a server
role and uses stock agent keys only: `wmi.getall`, `wmi.get`, `perf_counter_en`, `eventlog`. The agent
configuration stays as it is.

This file describes the English template, `template/template_windows_resource_exhaustion_en.yaml`. The
Russian version of the same template is `template/template_windows_resource_exhaustion.yaml`, described in
[README.md](README.md). Both files share UUIDs and the template name, so importing either one updates the
installed template in place and keeps its history. A server runs one of the two.

**Author:** Konstantin Tyutyunnik · [itforprof.com](https://itforprof.com) · MIT

## Why

The stock Windows templates (`Windows by Zabbix agent` and `... active`) collect nothing about sockets,
handles or paged pool, and nothing per process. Three common Windows server failures therefore stay
invisible until the server fails:

- **the dynamic port range runs out.** The server is up, but applications cannot connect. The culprit
  shows only in the Bound socket state, which neither `netstat` without `-q` nor `net.tcp.socket.count`
  reports;
- **one process accumulates handles.** Visible only per process;
- **nonpaged pool grows.** The counter is usually collected, but no stock template has a trigger on it.

Microsoft's guide to TCP port exhaustion says to run `netstat -anobq` and find the process with the most
BOUND entries. The template does this continuously.

## What is collected

24 items per host and one discovery rule. Four WMI queries run every five minutes: sockets, processes,
port range and physical memory. The raw socket and process tables are not stored; everything else is
computed by dependent items from one parse.

| Area | Metrics |
|---|---|
| Sockets | total, established, CLOSE_WAIT, TIME_WAIT, Bound - from `MSFT_NetTCPConnection` |
| Who holds them | the five processes with the most Bound and CLOSE_WAIT sockets, ready for operational data |
| Dynamic ports | free ports. The range is discovered from `MSFT_NetTCPSetting` |
| Handles | total, busiest process, top five as `name (PID n)=count` |
| Kernel pool | nonpaged and paged, in bytes and as a share of physical memory |
| Windows events | 4231 - no ephemeral port left; 4227 - local endpoint reused too fast |
| PID to name map | `PID=name` sorted by PID, to decode PIDs in socket alerts |

Three graphs: sockets by state, handles and kernel pool, free ports in the discovered range.

The template measures the port range itself. The discovery rule passes the start and size of the range to
the free-port item and its triggers, so nothing needs editing after `netsh int ipv4 set dynamicport tcp`.
Zabbix requires a discovery macro in a prototype key, so a new range creates a new item: its history starts
over and the forecast needs six hours of data again.

## Triggers

Every trigger is tied to the limit of a resource. There are no growth detectors: on 28 servers they
reported nine hosts at once while the nearest resource was 35 days from its limit. Handles then used 2.5%
of the per-process limit, ports 1.4% of the range, nonpaged pool 4.9% of RAM.

| Trigger | Severity | Fires when |
|---|---|---|
| Windows confirms the dynamic port range is exhausted | High | event 4231: no free port, outbound connections fail |
| Local endpoint reused faster than TIME_WAIT drains | Warning | event 4227: one application reconnects to one address too often. Widening the range does not help |
| Dynamic port range is running out | High | free ports below `{$WINEX.PORTS.FREE.PCT}`% of the discovered range |
| Dynamic port range will run out | Warning | the `timeleft` forecast on free ports is under `{$WINEX.PORTS.TIMELEFT}` |
| Connections are not being closed | Warning | CLOSE_WAIT sockets above the threshold for 15 minutes; the PID is in the operational data |
| Reconnect storm | Warning | TIME_WAIT sockets hold more than `{$WINEX.SOCK.TIMEWAIT.PCT}`% of the discovered range |
| Handles in one process are above the ceiling | Average | the busiest process's 24-hour minimum is above `{$WINEX.HANDLES.CEILING}`. Average: a standing count calls for a planned restart |
| Nonpaged pool above the ceiling | High | above `{$WINEX.POOL.NONPAGED.PCT}`% of physical memory |
| Paged pool above the ceiling | High | above `{$WINEX.POOL.PAGED.PCT}`% of physical memory |
| Socket probe: no data | Warning | the socket probe is silent while the agent is alive |
| Process probe: no data | Warning | the process probe is silent while the agent is alive |
| Kernel pool: no data | Warning | the pool counters are silent while WMI answers |
| Kernel pool ratio: no data | Warning | the pool ratios or the memory reading cannot be computed while the agent is alive |
| Port range probe: no data | Warning | the port range probe is silent while the agent is alive |

The no-data triggers fire only while the agent keeps sending the kernel pool counter, a 1-minute check
that does not use WMI. The counter must have arrived within the last 5 minutes and also one probe interval
earlier. The second check prevents false firings after a reboot, when the counter is back and the probe is
not yet. When the agent, proxy or link is down, the counter stops too and these triggers stay quiet; the
stock template reports that case. A replay over 31 days on 38 hosts gave two firings, both a real WMI
stall.

"Kernel pool: no data" cannot check the agent with its own counter, so it uses the WMI memory reading.

Each probe has its own no-data trigger, and child triggers depend only on it, so a stalled socket query
does not hide pool alerts whose data still arrives. While a probe is silent its child triggers keep their
state, and an open problem stays open.

Triggers that name a PID carry a "PID to name map" link in the problem.

## Requirements

- Windows 8 / Server 2012 or newer: older versions lack `root\StandardCimv2` with `MSFT_NetTCPConnection`.
- Zabbix agent or agent 2, 7.0 or newer, default configuration. Verified on agent 2 7.0.18 and Windows
  Server 2025. Agents older than 7.0 ignore the per-item timeout of an active check.
- `perf_counter_en`, not `perf_counter`: on a localized Windows the non-localized key finds nothing.
- Every item is an active check. The agent needs `ServerActive` pointing at the server or proxy and a
  `Hostname` matching the host in Zabbix. Port 10050 on the agent is not used.
- Link the stock **`Windows by Zabbix agent active`** to the same host. It is the only template that
  reports a dead agent or a wrong `ServerActive`.
  [zabbix-agent-config-guard](https://github.com/IT-for-Prof/zabbix-agent-config-guard) checks the
  setting itself.

## Installation

1. Import `template/template_windows_resource_exhaustion_en.yaml`
   (`Data collection → Templates → Import`, enable `Create new` and `Update existing`, leave
   `Delete missing` off).
2. Link it to Windows hosts together with the stock template.
3. After a day, go through the hosts where "Handles in one process are above the ceiling" fired and decide
   whether each holder is legitimate. If it is, override `{$WINEX.HANDLES.CEILING}` on that host.

The handle ceiling is the only threshold that may need an override. Ports and pool are shares of what the
host reports about itself: the discovered range and the physical memory. Windows has no such denominator
for handles. The normal count depends on the process: `dns.exe` holds about 10400 on any domain
controller, a healthy `lsass.exe` 2000-2500, the `searchd` indexer 40000-58000.

The handle trigger reads a 24-hour minimum, so it starts firing a day after linking and stays OK until
then.

## How problems close

Threshold triggers close when the value drops below the threshold. After a process restart the handle
problem closes within a day, once the low value becomes the minimum of the 24-hour window. The event name
keeps the count from the moment the problem opened; the operational data shows the current count.

Event-log triggers close after an hour without a new 4231 or 4227 event. An event-log item gets values only
during an incident, so without a recovery condition the problem would never close. No other trigger has a
recovery expression: where it can hold together with the problem expression, Zabbix opens and closes the
problem in a loop.

## Macros

| Macro | Default | Meaning |
|---|---|---|
| `{$WINEX.HANDLES.CEILING}` | `100000` | Handles the busiest process may hold all day, by its 24-hour minimum. The largest healthy minimum on the fleet was a search indexer at 58000 |
| `{$WINEX.NODATA.PERIOD}` | `15m` | How long a probe may stay silent before its no-data trigger fires. At least three `{$WINEX.PROBE.INTERVAL}` |
| `{$WINEX.POOL.NONPAGED.PCT}` | `20` | Nonpaged pool ceiling, % of physical memory |
| `{$WINEX.POOL.PAGED.PCT}` | `15` | Paged pool ceiling, % of physical memory |
| `{$WINEX.PORTS.FREE.PCT}` | `12` | Minimum free ports, % of the discovered range |
| `{$WINEX.PORTS.TIMELEFT}` | `1d` | How far ahead the port forecast looks |
| `{$WINEX.PROBE.INTERVAL}` | `5m` | Interval of the WMI probes, and the look-back of the agent liveness check in the no-data triggers |
| `{$WINEX.SOCK.BOUND.FLOOR}` | `500` | Bound sockets a host must hold for an hour before the port forecast may fire |
| `{$WINEX.SOCK.CLOSEWAIT.MAX}` | `200` | CLOSE_WAIT sockets on the host, held for 15 minutes |
| `{$WINEX.SOCK.TIMEWAIT.PCT}` | `25` | TIME_WAIT sockets, % of the discovered range |
| `{$WINEX.WMI.TIMEOUT}` | `30s` | Timeout of the WMI probes, the Zabbix maximum. Enumerating `MSFT_NetTCPConnection` takes about 4-5 ms per connection, so the socket probe stops fitting the timeout around 6000-7000 connections |

The pool ceilings are shares of memory because on x64 memory bounds the nonpaged pool. A byte ceiling would
be too tight on a large host and too loose on a very large one.

The port range is set with `netsh int ipv4 set dynamicport tcp start=N num=M`. Windows has ignored
`MaxUserPort` in the registry since Vista.

## What the template does not catch

- **A leak below the ceiling.** A process adding 1000 handles a day from 12000 is reported after 73 days,
  when it crosses `{$WINEX.HANDLES.CEILING}`. To watch a host earlier, lower the ceiling on that host.
- **A forecast for handles and pool.** `timeleft` needs history deeper than its horizon. Only ports have a
  forecast, with a one-day horizon.
- **A leak spread over a hundred processes** that does not move the busiest one. Such a leak raises the
  paged pool, and its ceiling fires.
- **Growing load versus a leak.** That is why the ceiling is set per host by someone who knows the server's
  role.
- **GDI and USER objects.** They are limited to 10000 per process and on terminal servers run out sooner
  than kernel handles.
- **RDS sessions** - the `\Terminal Services\*` counters.

## License

MIT. See [LICENSE](LICENSE).
