# Lab notes — LEO NTN testbed

Working notes for the LEO scenario. The GEO work, including the debugging that produced
the three OAI patches and the real-time scheduling requirement, is documented in the
[GEO repository](https://github.com/sabbir-uoulu/open-source-5g-ntn).

---

## Starting point

The LEO scenario reuses the GEO build unchanged: OAI tag `2026.w16` (commit `38dc378224`)
with all three patches applied, and the same Dockerised Open5GS core. No rebuild was
required. Everything below is configuration.

This matters because it means the hard problems were already solved in the GEO phase.
In particular, the real-time scheduling requirement (`sudo chrt -f 80` on both modems,
CPU governor set to `performance`) is orbit-independent: it addresses thread starvation
in the RF simulator's lock-step sample exchange, not the size of the delay.

---

## What shipped in-tree

OAI `2026.w16` already contains LEO configuration files:

- `ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-leo.conf`
- `ci-scripts/conf_files/nrue.uicc.ntn-leo.conf`
- `targets/PROJECTS/GENERIC-NR-5GC/CONF/channelmod_rfsimu_LEO_satellite.conf`

Both conf files already carried the LEO NTN parameters and, importantly, an inline
`channelmod` block plus `options = ("chanmod")` and `prop_delay = 20` in their
`rfsimulator` sections. Because the channel model is configured in-file rather than on
the command line, the command-line parsing problem encountered during the GEO work does
not arise for these options.

The LEO NTN values shipped correct and were left untouched:

```
ntn-UlSyncValidityDuration-r17 = 5;          # vs 240 for GEO
cellSpecificKoffset_r17        = 40;         # vs 478 for GEO
ta-Common-r17                  = 4627000;    # 18.84 ms
ta-CommonDrift-r17             = -230000;    # -46 µs/s   (absent in GEO)
positionY-r17                  = -2166908;
positionZ-r17                  =  4910784;
velocityVY-r17                 =  115246;    # 6914.76 m/s
velocityVZ-r17                 =   50853;    # 3051.18 m/s
prach_ConfigurationIndex       = 98;         # vs 7 for GEO
```

`disable_harq` is **absent** from the LEO gNB configuration, and that is correct rather
than an omission. Disabling HARQ feedback is a Release-17 measure for round-trip times
that make feedback impractical; at LEO's ~14 ms it works normally. The gNB confirms this
at startup with `HARQ feedback enabled, num DLHARQ:32, num ULHARQ:32`.

---

## Configuration changes made

Only the core-facing settings needed reconciling — the shipped files point at the stock
OAI demo core.

gNB (`gnb.sa.band254.u0.25prb.rfsim.ntn-leo.conf`):

| Setting | Shipped | Changed to |
|---|---|---|
| `tracking_area_code` | 1 | 7 |
| `plmn_list` | mcc 208 / mnc 99, sst 1 + sd 0xffffff | mcc 1 / mnc 1, sst 1 (no sd) |
| `amf_ip_address` | 192.168.71.132 | 172.22.0.10 |
| `GNB_IPV4_ADDRESS_FOR_NG_AMF` | 192.168.71.140/26 | 172.22.0.1/24 |
| `GNB_IPV4_ADDRESS_FOR_NGU` | 192.168.71.140/26 | 172.22.0.1/24 |

UE (`nrue.uicc.ntn-leo.conf`):

| Setting | Shipped | Changed to |
|---|---|---|
| `imsi` | 208990100001100 | 001010000000001 |
| `dnn` | oai | internet |

The shipped `key` and `opc` already matched the provisioned subscriber.

---

## Run commands

Core first, then both modems under real-time scheduling. The four trailing UE options are
LEO-specific and have no GEO equivalent.

```bash
# gNB
sudo chrt -f 80 env RFSIMULATOR=server ./ran_build/build/nr-softmodem \
  -O ../ci-scripts/conf_files/gnb.sa.band254.u0.25prb.rfsim.ntn-leo.conf \
  --rfsim --gNBs.[0].min_rxtxtime 6

# UE
sudo chrt -f 80 env RFSIMULATOR=127.0.0.1 ./ran_build/build/nr-uesoftmodem \
  -O ../ci-scripts/conf_files/nrue.uicc.ntn-leo.conf \
  --band 254 -C 2488400000 --CO -873500000 \
  -r 25 --numerology 0 --ssb 60 --rfsim \
  --time-sync-I 0.1 --ntn-initial-time-drift -46 \
  --initial-fo 57340 --cont-fo-comp 2
```

`--ntn-initial-time-drift` and `--initial-fo` give the UE the drift and frequency-offset
starting values it needs before SIB19 is decoded. `--cont-fo-comp 2` enables continuous
Doppler compensation: post-compensation on the downlink, pre-compensation on the uplink.

---

## Result

The attach succeeded on the first run. No debugging was required, which is a direct
consequence of the GEO phase having already resolved the scheduling and RAR-window issues.

Channel model confirmed active on both directions:

```
[OCM] Model rfsimu_channel_enB0 type SAT_LEO_TRANS allocated from config file
[OCM] Model rfsimu_channel_ue0 type SAT_LEO_TRANS allocated from config file
SAT_LEO_TRANS: satellite orbit height 600.000000 km
```

Random access, with the UE's timing-advance pre-compensation exact:

```
[RAPROC] 511.4 Initiating RA procedure with preamble 58, energy 40.0 dB
UE b801: 512.1 Generating RA-Msg2 DCI, RAPID 58, timing_offset = 0 (estimated distance 0.0 m)
[RAPROC] Received SDU for CCCH length 6 for UE b801
Activating scheduling Msg4 for TC_RNTI 0xb801
```

The `timing_offset = 0` is worth noting: the residual timing error at the gNB after the
UE's NTN pre-compensation was zero.

Core side:

```
amf: [imsi-001010000000001] Registration complete
smf: UE SUPI[imsi-001010000000001] DNN[internet] IPv4[192.168.100.3]
```

Data plane: `oaitun_ue1` at 192.168.100.3, ICMP to 8.8.8.8 with 0% loss.

---

## Orbital tracking across a pass

The distinguishing LEO result. Sampled across 214 PHY records from one run:

| Phase | Delay | DL Doppler | TA (samples) | Drift |
|---|---|---|---|---|
| Rising | 18.82 ms | +57.34 kHz | 289,095 | −46.09 µs/s |
| | 13.42 ms | +56.65 kHz | 206,041 | −45.53 µs/s |
| | 8.21 ms | +52.19 kHz | 126,039 | −41.96 µs/s |
| **Closest approach** | **4.22 ms** | **≈ 0, crossing** | **65,626** | **sign change** |
| | 5.79 ms | −43.27 kHz | 88,903 | +34.80 µs/s |
| Receding | 8.07 ms | −51.93 kHz | 124,034 | +41.74 µs/s |

The delay reaches a minimum at closest approach and rises again; the Doppler crosses zero
and reverses sign. The `variant` field (drift acceleration) peaks near the crossing, where
the geometry changes fastest.

SIB19 is re-read roughly every 2.5 s:

```
[NR_RRC] Found SIB19
[NR_RRC] val430:5 s, T430:5000 ms, sib19_timer:2500 ms
```

against GEO's 240 s validity — the UE cannot coast on stale ephemeris in LEO.

---

## Observations

**The GEO RAR-window patch is not needed at LEO.** A GEO round trip (~477 ms) exceeds the
base `ra-ResponseWindow` maximum of 80 ms, so the window expired before the RAR could
arrive. A LEO round trip (~14 ms) fits comfortably. The patch is still applied here and is
harmless — it adds the K-offset, 40 slots — but this confirms it addresses a GEO-scale
problem specifically rather than NTN generally.

**HARQ feedback is orbit-dependent.** Enabled at LEO, disabled at GEO. Worth stating,
because "NTN disables HARQ" is a common oversimplification.

**IPv6 router solicitations produce UPF errors.** The UPF logs
`Invalid packet [IP version:6, Packet Length:48]` with payloads beginning `fe80::`. This is
the UE's IPv6 link-local traffic on a session provisioned as IPv4-only. It does not affect
the IPv4 data path. Disable IPv6 on `oaitun_ue1`, or provision the session as IPv4v6, to
silence it.

**Wall-clock latency still does not reflect the modelled delay**, for the same reason as
GEO: the RF simulator is not real-time. The orbital tracking above is the meaningful
evidence, and it is unaffected by this limitation because it lives in the sample/timing
domain rather than in wall-clock.

---

## Toward multi-satellite handover

Investigated but not yet implemented. Findings so far:

- The two-cell rfsim setup documented in `doc/handover-tutorial.md` requires the
  **RF simulator roles to be reversed**: the UE acts as the server and both DUs connect as
  clients, because the simulator supports one server with multiple clients.
- The OAI UE **does not support measurement reporting**, so A3/RSRP-triggered handover is
  unavailable. Handover must be triggered externally, via the telnet interface
  (`ci trigger_f1_ho`). This is a limitation, but it also means an ephemeris- or
  timing-advance-driven trigger is the natural approach rather than merely a better one.
- OAI's own NTN handover documentation lists five candidate trigger criteria for LEO —
  measurement, location, elevation angle, time, and timing advance — states that
  measurement-based triggering is inefficient for LEO, and implements none of them beyond
  a manual trigger.
- The `SAT_LEO_TRANS` model **hardcodes the starting orbital phase** to the point where the
  satellite rises above the horizon, so two channel instances describe the same satellite.
  Orbital position derives from a per-connection `start_TS`, so staggering the DU start
  times produces a phase separation; a ~6.4 minute stagger gives roughly half a pass. A
  configurable phase offset would make this reproducible and is the cleaner fix.

For reference, at 600 km: orbital period 96.7 minutes, visible arc 47.9°, visible pass
duration approximately 771 s (12.9 minutes).
