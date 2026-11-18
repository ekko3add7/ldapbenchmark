# RHDS 13 (389-ds-base 3.3.0): requests on one connection sometimes wait exactly 250 ms before the server reads them

## 1. Scenario

We are running a performance comparison of RHDS and OpenLDAP. One test sends user lookups
that look like what SSSD sends for `getpwuid()`:

```
base:   ou=People,dc=ldapbench,dc=lab
scope:  sub
filter: (&(uidNumber=<N>)(objectClass=posixAccount))
attrs:  uid uidNumber gidNumber cn homeDirectory loginShell
bind:   anonymous
```

To measure the latency of a single request, a client opens **one connection** and sends the
searches **one at a time**: the next request is sent only after `SearchResultDone` of the
previous one is received. The server has no other load.

| Item | Value |
|---|---|
| Server | rhds13-m1: VMware VM, 8 vCPU Intel Xeon Silver 4314, 15 GB RAM |
| OS | Red Hat Enterprise Linux 10.2, kernel 6.12.0-211.73.1.el10_2.x86_64 |
| Packages | 389-ds-base-3.3.0-5.el10_3, nss-3.124.0-5.el10_0, nspr-4.39.0-5.el10_0 |
| Instance | `slapd-ldapbench`, 2-node multi-supplier replication (rhds13-m1 / rhds13-m2) |
| Data | 100,000 users, all in the entry cache; `uidNumber` and `objectClass` equality indexes |
| Server config | Defaults: `nsslapd-enable-turbo-mode: on`, `nsslapd-numlisteners: 1`, `nsslapd-threadnumber: 16`, `nsslapd-maxthreadsperconn: 5`, `nsslapd-accesslog-logbuffering: on` |
| Client | ldapclient-0: RHEL 8.10, 2 vCPU, same subnet, ping RTT ~0.25 ms |
| Reference | OpenLDAP 2.6.10 on Ubuntu 24.04, same VM size, same data, same client |

## 2. Problem

About **0.1% to 0.3% of the requests take ~250 ms** instead of ~0.2 to 0.4 ms.

- The slow requests are **always 250 to 252 ms**. Never 50 ms, never 1 s.
- The server **starts** these operations ~250 ms late. The operation itself is fast
  (`etime` < 1 ms).
- Because 0.15% x 250 ms is large, the **average** latency is about 2x to 4x the median,
  and a run of 10,000 searches with `ldapsearch -f` spends about **half of its time** waiting
  (3.5 s of 6.1 s, see section 6).
- OpenLDAP shows no such delay with the same client and the same requests.

## 3. Findings

**3.1 Not network, not TLS, not the client.** It happens on every transport, also when the
client runs on the server host (`evidence/05_transport_comparison.txt`, 5,000 searches each):

| Client location | Protocol | Requests > 100 ms | p50 | max |
|---|---|---:|---:|---:|
| ldapclient-0 (network) | LDAPS 636 | 15 (0.30%) | 0.371 ms | 251.3 ms |
| rhds13-m1 itself | LDAPS 636 | 14 (0.28%) | 0.225 ms | 250.8 ms |
| rhds13-m1 itself | LDAP 389 (no TLS) | 13 (0.26%) | 0.193 ms | 250.8 ms |
| rhds13-m1 itself | LDAPI (Unix socket) | 15 (0.30%) | 0.179 ms | 251.0 ms |
| Reference: ldapclient-0 to OpenLDAP | LDAPS 636 | **0 (0%)** | 0.349 ms | 4.0 ms |

It also happens with the standard `ldapsearch -f` (section 6) and with python-ldap3.

**3.2 The delay is before the operation starts.** In the access log, the time between
`RESULT` of op N-1 and `SRCH` of op N is ~250 ms, but `wqtime`, `wtime`, `optime` and `etime`
of op N are all below 1 ms (section 4).

**3.3 Traffic on another connection hides it.** With a second connection sending one search
every 2 ms, the delay almost disappears (`evidence/06_background_traffic.txt`):

| rhds13-m1, LDAPI, 5,000 searches | Requests > 3 ms | Average |
|---|---:|---:|
| Only the test connection | 11 (0.22%), all >= 200 ms | 0.751 ms |
| Plus one connection sending every 2 ms | 1 (0.02%) | 0.236 ms |

So the waiting request is picked up as soon as the server wakes up for any reason.

**3.4 Settings do not remove it.** `evidence/07_turbo_numlisteners.txt`, 10,000 searches per
cell, `dirsrv` restarted for every variant, settings restored after:

| Variant | Local LDAPI | Local LDAP 389 | Client LDAPS |
|---|---:|---:|---:|
| turbo `on`, numlisteners `1` (default) | 0.14% | 0.15% | 0.15% |
| turbo `off`, numlisteners `1` | 0.25% | 0.37% | 0.28% |
| turbo `on`, numlisteners `4` | 0.17% | 0.17% | 0.15% |
| turbo `off`, numlisteners `4` | 0.92% | 0.25% | 0.23% |

The max delay is 250.8 to 251.2 ms in every variant. Turbo mode `off` makes it about 2x worse.
numlisteners has no effect. (Single run per cell, about +/-30% noise.)

**3.5 Not access log rotation or access log writing.** Access log rotates at 100 MB or once a
day. No rotation happened during any of the delay tests (last rotations 08:52:33 and 09:46:39
local time; tests at 08:54-08:55, 09:09-09:10, 09:55, 09:57). With the access log **disabled**,
the delay is still there (`evidence/10_accesslog_rotation_and_off.txt`, LDAPI, 10,000 searches):

| Access log | Run 1 | Run 2 | Delay length |
|---|---:|---:|---|
| on (default) | 14 (0.14%) | 16 (0.16%) | 251 ms |
| off | 15 (0.15%) | 21 (0.21%) | 251 ms |

**3.6 Also on RHDS 11.** rhds11-m1 (389-ds-base-1.4.3.31) shows the same exact 250 ms delays
at a similar rate.

## 4. Log evidence

From the run with `ldapsearch -f` over LDAPS from ldapclient-0
(`evidence/02_reproduce_ldapsearch_ldaps_client.txt`). Access log of rhds13-m1 (local time, -0500):

```
[16/Nov/2026:09:57:02.540942470 -0500] conn=26 op=860 SRCH base="ou=People,dc=ldapbench,dc=lab" scope=2 filter="(&(uidNumber=108616)(objectClass=posixAccount))" attrs="uid"
[16/Nov/2026:09:57:02.541053536 -0500] conn=26 op=860 RESULT err=0 tag=101 nentries=1 wqtime=0.000037165 wtime=0.000103941 optime=0.000111137 etime=0.000214914
[16/Nov/2026:09:57:02.791502289 -0500] conn=26 op=861 SRCH base="ou=People,dc=ldapbench,dc=lab" scope=2 filter="(&(uidNumber=111084)(objectClass=posixAccount))" attrs="uid"
[16/Nov/2026:09:57:02.791838123 -0500] conn=26 op=861 RESULT err=0 tag=101 nentries=1 wqtime=0.000066081 wtime=0.000203668 optime=0.000336686 etime=0.000540170
```

- op 860 finished at `.541053`. `ldapsearch` sends op 861 right away.
- op 861 starts at `.791502`: **250.4 ms** later. Its `etime` is only 0.54 ms.
- This run: 10,000 searches, **12 gaps of ~250 ms** (0.12%), total run time 8.4 s.

The same pattern with per-request client timing (`evidence/03_access_log_gaps_ldaps_client_3conns.txt`,
`evidence/09_latency_run_per_request/`). The client measured 251.2 ms for `uidNumber=144232`:

```
[16/Nov/2026:08:55:02.112060413 -0500] conn=899 op=300 RESULT err=0 tag=101 nentries=1 wqtime=0.000026021 wtime=0.000097590 optime=0.000113663 etime=0.000211089
[16/Nov/2026:08:55:02.362599185 -0500] conn=899 op=301 SRCH base="ou=People,dc=ldapbench,dc=lab" scope=2 filter="(&(uidNumber=144232)(objectClass=posixAccount))" attrs="uid uidNumber gidNumber cn homeDirectory loginShell"
[16/Nov/2026:08:55:02.362992941 -0500] conn=899 op=301 RESULT err=0 tag=101 nentries=1 wqtime=0.000084376 wtime=0.000271804 optime=0.000396559 etime=0.000668147
```

On this connection the normal gap between `RESULT` and the next `SRCH` is 0.32 ms (p50);
6 gaps were 250.4 to 250.8 ms, and they match the 6 slow requests seen by the client one to one.

Summary of all access log checks (normal gap = p50 of `RESULT` op N-1 -> `SRCH` op N):

| Run | Connection | Searches | Normal gap | Gaps > 100 ms | Max gap | Evidence |
|---|---|---:|---:|---:|---:|---|
| ldapsearch, localhost LDAP 389 | conn=24 | 10,000 | - | 14 | 250.6 ms | `01` |
| ldapsearch, client LDAPS | conn=26 | 10,000 | - | 12 | 250.4 ms | `02` |
| client LDAPS (raw requests) | conn=899 | 2,000 | 0.32 ms | 6 | 250.8 ms | `03` |
| client LDAPS (python-ldap3) | conn=900 | 2,000 | 1.03 ms | 7 | 250.6 ms | `03` |
| localhost LDAP 389 | conn=928 | 5,000 | 0.11 ms | 13 | 250.5 ms | `04` |
| localhost LDAPI | conn=929 | 5,000 | 0.11 ms | 15 | 250.7 ms | `04` |

## 5. Suspected cause: `SLAPD_WAKEUP_TIMER` (250 ms)

The binary on rhds13-m1 uses `poll` (no `epoll_*` symbols in `ns-slapd` / `libslapd.so.0`), so the
non-epoll code path below is the one that runs. References are for tag `389-ds-base-3.3.0`.

1. The `ct-list` thread rebuilds the poll set and waits with a **fixed 250 ms timeout**:
   [daemon.c L78](https://github.com/389ds/389-ds-base/blob/389-ds-base-3.3.0/ldap/servers/slapd/daemon.c#L78)
   `slapd_ct_thread_wakeup_timer = SLAPD_WAKEUP_TIMER` (250),
   [daemon.c L1565-L1574](https://github.com/389ds/389-ds-base/blob/389-ds-base-3.3.0/ldap/servers/slapd/daemon.c#L1565-L1574)
   `setup_pr_read_pds()` then `POLL_FN(..., pr_timeout)`. There is no `cn=config` attribute for it.
2. While building the poll set, a connection whose `c_mutex` is busy is **skipped**:
   [daemon.c L1840](https://github.com/389ds/389-ds-base/blob/389-ds-base-3.3.0/ldap/servers/slapd/daemon.c#L1840)
   `if (pthread_mutex_trylock(&(c->c_mutex)) == EBUSY) { c = next; continue; }`
3. After a worker reads a request, it makes the connection readable and wakes the poll thread:
   [connection.c L1988-L1992](https://github.com/389ds/389-ds-base/blob/389-ds-base-3.3.0/ldap/servers/slapd/connection.c#L1988-L1992)
   `connection_make_readable_nolock(conn)` ... `signal_listner(conn->c_ct_list)`.
4. When the operation finishes, a normal connection (not replication, not timed out, not at max
   threads) does **not** wake the poll thread again:
   [connection.c L2142-L2174](https://github.com/389ds/389-ds-base/blob/389-ds-base-3.3.0/ldap/servers/slapd/connection.c#L2142-L2174)
   `need_wakeup` stays 0.

Possible sequence for one delayed request:

```
worker:  read op N, make conn readable, signal_listner()          (3)
ct-list: wake up, setup_pr_read_pds()
worker:  holds c_mutex (processing / sending result / removing op N)
ct-list: trylock(c_mutex) == EBUSY -> conn skipped                  (2)
ct-list: poll() without this conn, timeout 250 ms                   (1)
worker:  op N done, releases conn, no signal_listner()              (4)
client:  gets result N, sends op N+1 at once -> data waits in the socket
ct-list: 250 ms timeout -> rebuild poll set -> sees op N+1 -> op N+1 starts ~250 ms late
```

This matches all findings: the exact 250 ms, no relation to logging, the low rate (the lock must be busy at that
moment), small `wqtime/wtime/optime` for the delayed operation, all transports affected, and
other connections hiding it (they wake the poll thread). Turbo mode reduces it because a worker
in turbo mode reads the next request directly and does not go back through the poll set.
We did not change the code to prove this.

**Questions**

1. Is this sequence correct? Is the delay expected behavior?
2. When `setup_pr_read_pds()` skips a connection because `trylock` fails, should something wake
   the poll thread again, for example `signal_listner()` at the end of an operation
   (connection.c L2142-L2174)?
3. Is this a known issue (bug / Jira / upstream)? Is it fixed in a newer version, or does the
   `epoll` build (`--enable-epoll`) avoid it?
4. Is there a supported way to avoid it? `SLAPD_WAKEUP_TIMER` is a compile-time constant, and
   `nsslapd-enable-turbo-mode` / `nsslapd-numlisteners` do not remove it (section 3.4).
5. Is there extra logging or tracing we can turn on to show where the request waits?

## 6. How to reproduce

Only standard tools: `ldapsearch` (openldap-clients) and `awk`. Run on the server itself to
rule out the network. Adjust the base DN and the uidNumber range to your data.

**Step 1: send 10,000 searches on one connection.** `ldapsearch -f` uses one connection and
sends the requests one after another; `%s` is replaced by each line of the file.

```bash
seq 50001 150000 | shuf -n 10000 > /tmp/uids.txt
time ldapsearch -x -H ldap://localhost:389 -b ou=People,dc=ldapbench,dc=lab \
  -f /tmp/uids.txt "(&(uidNumber=%s)(objectClass=posixAccount))" uid > /dev/null
```

Also try `-H ldapi://%2frun%2fslapd-INSTANCE.socket` or `-H ldaps://HOST:636` from a client.
In our runs, 10,000 searches took 6.1 s (localhost) and 8.4 s (LDAPS from a client), and
3.0 to 3.5 s of that was the 250 ms waits.

**Step 2: wait ~15 s** (the access log is buffered), then **find the gaps** between `RESULT`
of op N-1 and `SRCH` of op N on that connection:

```bash
L=/var/log/dirsrv/slapd-INSTANCE/access
CONN=$(grep "connection from 127.0.0.1" $L | tail -1 | grep -o "conn=[0-9]*")
awk -v c="$CONN " 'index($0, c) && / (RESULT|SRCH) / {
    split($1, a, ":"); t = a[2]*3600 + a[3]*60 + a[4]
    if ($0 ~ / SRCH /) { if (last) { g = (t - last) * 1000; n++; if (g > 100) { s++; printf "gap %.1f ms before %s %s\n", g, $3, $4 } } }
    else last = t
  } END { printf "requests: %d, gaps > 100 ms: %d (%.2f%%)\n", n, s, s * 100 / n }' $L
```

For a client over LDAPS, replace `127.0.0.1` with the client IP.

**Expected output** (`evidence/01_reproduce_ldapsearch_localhost.txt`):

```
gap 250.5 ms before conn=24 op=17
gap 250.5 ms before conn=24 op=881
...
gap 250.4 ms before conn=24 op=5362
requests: 10000, gaps > 100 ms: 14 (0.14%)
```

**Optional checks**

- Start a second `ldapsearch -f` with a large file in the background: the number of gaps on the
  first connection goes down a lot (finding 3.3).
- Set `nsslapd-enable-turbo-mode: off`: the number of gaps goes up (finding 3.4).

## Attachments

| File | Content |
|---|---|
| `sosreport/sosreport-rhds13-m1-ldapbench-250ms-stall-2026-11-16-iuomwpa.tar.xz` | sosreport of rhds13-m1, sha256 `fd4e9c0f4cd52d9d37a4b4e99776ff0cc01875a6aa443eef4de2027e032cb128` |
| `evidence/01_reproduce_ldapsearch_localhost.txt` | Section 6 on rhds13-m1, LDAP 389 on localhost: 14 gaps / 10,000 |
| `evidence/02_reproduce_ldapsearch_ldaps_client.txt` | Section 6 from ldapclient-0 over LDAPS: 12 gaps / 10,000, with access log lines |
| `evidence/03_access_log_gaps_ldaps_client_3conns.txt` | Access log lines around every gap, LDAPS from ldapclient-0 (conn=898 warm-up, conn=899 raw requests, conn=900 python-ldap3) |
| `evidence/04_access_log_gaps_local_ldap_ldapi.txt` | Same for localhost LDAP 389 and LDAPI |
| `evidence/05_transport_comparison.txt` | Finding 3.1: results for each transport, with every slow request and its send time (UTC) |
| `evidence/06_background_traffic.txt` | Finding 3.3 |
| `evidence/07_turbo_numlisteners.txt` | Finding 3.4 |
| `evidence/08_rhds13_config.txt` | Connection-related `cn=config` values and package versions |
| `evidence/10_accesslog_rotation_and_off.txt` | Finding 3.5: rotation times vs test times, and access log on / off |
| `evidence/09_latency_run_per_request/` | Client time for every request (`client/latency_ops.csv`) and the matching access log (`m1/server.log`) |

Test times (UTC, 2026-11-16): per-request run 13:54:58 - 13:55:08; transport comparison
14:09:50 - 14:10:40; ldapsearch reproduction ~14:55 (localhost) and 14:57:02 (LDAPS).
