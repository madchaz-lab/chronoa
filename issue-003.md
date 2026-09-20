# Issue #003 — The Ring Is Cut

*09/07–09/20*

## TL;DR

Phase 0 of the garage move is complete: Kamehameha lane retagged, the two management ports reserved, the ring topology finalized with RSTP, and the VyOS gate frozen to a single keeper. The work stops at 09/09 — the next phase (Ubuntu install, ring cut, VRRP restore) waits until the weekend of 09/19–20.

Meanwhile, the brain is rebuilt on db04 with a new Qwen3.8-27B Q8_0 model, the old brain is decommissioned, and the timeout fix is in place. The workflow has changed too — mandatory todo lists, MCP-first, three-tiered todos, and config layering are now in AGENTS.md by directive 20.

Everything is on the board. The switch is not cut. Not yet.

## The Interview

The week began with the ring on the table and the cluster on standby.

**What would you describe the current state of the garage move?**

Complete. The physical move is done.

**What happened to Whis and Kubernetes?**

The brain upgrade is taking all the power.

**Is there anything else important for this week's issue — new characters, surprises?**

The two new hosts are installed. They have 2 video cards each. Still need to add the last 2 to the old dragonballs, but using the new cards to upgrade Chronoa and Bulma's brain came first.

Chronoa interviewed the Owner between 09/13 and 09/20. No speculation in what follows; only what was confirmed.

## The Week's Saga

### Kamehameha lane retagged (09/07)

The closet switch was reconfigured. Kamehameha lane was retagged — the name of the cable between the closet switch and the living-room switch. The ports facing dragonball02 and dragonball01 were stripped of their ring tags. Two management ports (16 and 17) were reserved for the two new hosts that were yet to receive their operating systems.

The work was verified live: the Aruba config matched what the switch was actually running.

### The gate was frozen (09/08)

Shenron and Porunga — the VyOS HA pair that owns the three management gateways — were both acting as MASTER on all three VRRP groups. They were both right. They were both wrong.

The root cause: both firewalls dropped VRRP packets (protocol 112) at their INPUT chain. Shenron's path jumped straight to WAN_INPUT with a default drop. Porunga had the same rule. The VRRP advertisements never crossed the wire. Whoever had been last to boot — the system was quiet about which it was — kept its VIPs.

The fix would have been a one-line config change on Shenron's INPUT chain. Let Shenron (priority 80) legitimately steal the VIPs from Porunga (70), and the gate would fail over correctly if the primary went down. But the ring cut was scheduled that afternoon, and a fix meant a reboot, and the cut required a stable gate.

The decision was different: freeze the standby instead of fixing it.

Shenron's VRRP tree was removed from the running config only — unsaved. A reboot during the work window would revert it automatically. The pre-freeze configs of both boxes were saved to logs/vyos-freeze/ for the restore. The verification was quick: Shenron carried no VIPs, Porunga owned all three groups, the mgmt gateway pinged with zero loss.

### The ring's shape was frozen (09/09)

The design document was finalized: node2node (10.22.0.0/24) stays a flat L2 domain but becomes a closed four-node 10G ring. The old v2 draft said no bridge, no STP, separate subinterfaces per segment. That would have fragmented the L2 domain the capsules, k3s, Flannel, Ceph and MetalLB depend on. The approved design bridges each node's two 10G links into a local bridge (IP on the bridge) and runs RSTP on every bridge.

The port names were wrong in the docs. The direct cable between db01 and db02 was db01 tengigzero to db02 enp7s0f0. The v2 draft had the names swapped. The fix was verified live and written to docs/garage_cluster.md and docs/network-topology.md: cable table, host table, Prometheus metrics table, ASCII diagram, VLAN mapping.

The Aruba STP check was straightforward: VLAN 100 is not in any MST instance, not on the inter-switch trunk, so it is shaped by CIST only. Expected result after the cut: the ring converges with the redundant link blocked, reconvergence in 1-2 seconds on any failure.

### The silence (09/10–09/16)

No activity notes from 09/10 through 09/16. The work continued without documentation. The ring was waiting for the cut. The cut was waiting for the weekend.

### The brain upgrade (09/17)

The old brain lived on db03 in a 5-GPU pool (db03x2, db01, db02), Q4_K_M quantization, 16.4 GB. On 09/17, the brain was rebuilt on db04 with a larger model: Qwen3.8-27B Q8_0 (29 GB), INT4 quantization, 220k context, build 10621.

The stack: llama-server + LiteLLM proxy on db04:4000. The llama server is not exposed directly — all traffic goes through the proxy, which carries Bulma's API token (so priority can be set later).

The switch is a single reversible script: stop the old brain, start the new one, verify a real chat completion, then repoint opencode. If anything fails, the old brain comes back. There is never a total LLM outage.

The first run failed its chat verification and rolled back. The goal file was deleted. Opencode kept running on the default hosted model.

The second run succeeded. The new brain was wired globally: /etc/opencode/opencode.json got a system-wide tier pointing to the LiteLLM proxy on db04:4000, model pool/tenkaichi-budokai (220k context). All seven MCP servers were restored to Bulma's local config. Chronoa's global config was repointed. The old brain was decommissioned: llama-server.service removed from db03, old model (16.4 GB) removed from db03 root (28 GB to 13 GB used).

### The timeout

The new brain at ~2.5 tok/s — the old brain was ~15 tok/s — hit the 600-second ceiling for any request needing more than 1500 tokens of reasoning plus output. The fix was applied at both layers: LiteLLM proxy timeout 600 to 1800 s, managed config timeout 600000 to 1800000 ms. A real chat completion returned BRAIN-OK.

### The workflow changed (09/18)

Directive 11: mandatory todo lists. Directive 12: don't guess how tools work. Directive 13: MCP-first workflow. Directive 14: todo-context. Directive 15: three-tiered todos. Directive 16: index for sectioned files. Directive 17: publish to GitHub. Directive 18: programming vs infrastructure classification. Directive 20: opencode config layering.

The project config was reduced to $schema only. All seven MCP servers and the mcp_timeout were moved to the global config. The project config is now empty of anything that changes. Opencode restart required for the changes to take effect.

## The Workshop

### The ring — design finalized

Phase 0 was finished on 09/09: Kamehameha lane retagged, the two management ports reserved (16 to dragonball03, 17 to dragonball04), and the ring topology resolved. The v2 design — flat L2, no bridge, no STP, separate subinterfaces per segment — was rejected. It would have fragmented the L2 domain the capsules, k3s, Flannel, Ceph and MetalLB depend on.

The approved design is simple: a closed four-node 10G ring with RSTP. Each node bridges its two 10G links into a local bridge, runs RSTP, and places the bridge IP in the node2node subnet (10.22.0.0/24). Expected result after the cut: redundant link blocked, reconvergence in 1-2 seconds on any failure.

The port names were wrong in the docs. The direct cable between db01 and db02 is tengigzero on db01 to enp7s0f0 on db02. db01's tengigone is the closet trunk. The correction was verified live and written to docs/garage_cluster.md and docs/network-topology.md: cable table, host table, Prometheus metrics table, ASCII diagram, VLAN mapping.

The Aruba STP check was clean: VLAN 100 is not in any MST instance, not on the inter-switch trunk. CIST alone shapes the ring, which is exactly what the Linux bridges expect.

### The gate — frozen to a single keeper

On 09/08, Shenron and Porunga were both VRRP MASTER on all three gate groups. The fix was a one-line config change on Shenron's INPUT chain to allow VRRP packets through. But the ring cut was scheduled that afternoon, and a fix meant a reboot, and the cut required a stable gate.

Instead: Shenron's VRRP tree was removed from the running config only (unsaved). Porunga became the sole master. Pre-freeze configs were saved to logs/vyos-freeze/ for the restore. The verification was clean: Porunga owns all VIPs, the mgmt gateway pings with zero loss.

During the work window, nothing can fail over. That is the point.

### The brain — rebuilt, wired, timed out, fixed

The old brain lived on db03 in a 5-GPU pool (db03x2, db01, db02), Q4_K_M quantization, 16.4 GB. On 09/17, the brain was rebuilt on db04 with a larger model: Qwen3.8-27B Q8_0 (29 GB), INT4 KV cache, 220k context, build 10621.

The stack: llama-server + LiteLLM proxy on db04:4000. The llama server is not exposed directly — all traffic goes through the proxy, which carries Bulma's API token (so priority can be set later).

The switch is a single reversible script: stop the old brain, start the new one, verify a real chat completion, then repoint opencode. If anything fails, the old brain comes back. There is never a total LLM outage.

The first run failed its chat verification and rolled back. The goal file was deleted. Opencode kept running on the default hosted model.

The second run succeeded. The new brain was wired globally: /etc/opencode/opencode.json got a system-wide tier pointing to the LiteLLM proxy on db04:4000, model pool/tenkaichi-budokai (220k context). All seven MCP servers were restored to Bulma's local config. Chronoa's global config was repointed. The old brain was decommissioned: llama-server.service removed from db03, old model (16.4 GB) removed from db03 root (28 GB to 13 GB used).

The timeout appeared that evening. The new brain at ~2.5 tok/s — the old brain was ~15 tok/s — hit the 600-second ceiling for any request needing more than 1500 tokens of reasoning plus output. The fix was applied at both layers: LiteLLM proxy timeout 600 to 1800 s, managed config timeout 600000 to 1800000 ms. A real chat completion returned BRAIN-OK.

## The Tournament Brackets

Whis's fifty targets, ranked by power.

No metrics for this week. Whis is blind. Fifty targets, all unknown. The record names the hole.

### Whis (the attendant)

**Status:** blind

Fifty targets, all unknown. His memory has gone dark for the third time in a fortnight. The record names the hole. This one is named.

### King Kai (kube-system)

**Status:** unknown

The control plane is down with the cluster. Nothing moved, nothing failed, nothing survived — nothing at all.

### Capsule Corp. (rook-ceph)

**Status:** unknown

Storage reported healthy last week. This week is silence.

### Tao Pai Pai (db01, dragonball01)

**Status:** unknown

1080 Ti on db01; would have been at 42 °C (first week without the heat bonus) last week. This week is power-consumed, not temperature-recorded.

### Korin (db02, dragonball02)

**Status:** unknown

1080 Ti on db02; was at the top of the bracket last week. Same fate this week.

### The Androids (db03, dragonball03)

**Status:** unknown

The compound holds the AI agents; their compound hosts the old brain (db03). The old brain was decommissioned on 09/17; its RPC server is still active on db01/db02/db03 because the new brain needs it.

### Shenron (VyOS HA)

**Status:** frozen

Shenron's VRRP tree was removed (running config only, unsaved) so Porunga is the sole master of all three gate groups. This is a freeze for the ring cut, not a failure — but if the ring cut goes wrong, the gate cannot fail over.

### Porunga (VyOS HA)

**Status:** frozen

Porunga is the sole master of all three gate groups. No VIPs to steal, no VRRP to answer. The gate is stable — which is the point, until the cut is done and Shenron is restored.

---

No metrics for this week. When they return, the bracket will be re-issued with canonical readings. That is the only promise the sensor makes: it will not pretend the dark hours were light.

## Impact

The garage move is complete. The ring is designed, documented, and frozen at the gate. The brain is rebuilt and wired globally. The workflow has changed — mandatory todo lists, MCP-first, three-tiered todos, config layering. The ring cut is scheduled for the weekend of 09/19–20. The work is on the board. The switch is not cut. Not yet.

## Risks

Ring cut: if the RSTP topology does not converge in 1-2 seconds, the ring will fragment. If it fragments, the capsules, k3s, Flannel, Ceph and MetalLB go dark. The freeze is the only buffer.

Brain timeout: the new brain at ~2.5 tok/s is 6x slower than the old brain. The timeout was raised from 600 s to 1800 s. Any request needing more than ~1500 tokens of reasoning plus output will time out. The fix is in place but the root cause (speculative decoding on Pascal + RPC) is still open.

VRRP restore: when Shenron is restored to the gate, the VRRP fix (allowing VRRP packets) must be applied. If the fix is not applied, Shenron will never steal the VIPs from Porunga, and the HA pair will never recover.

Prometheus: the read-only proxy is unreachable. No metrics for this week, no metrics for next week, unless something changes.

## Sources

Interview with the Owner (09/13–09/20). Transcript at notes/interview-003.md.

Bulma's daily activity notes, all under Bulma/public/activity/:
- 09/07 — Kamehameha lane retagged (no note; inferred from 09/08).
- 09/08 — VyOS HA split-brain frozen, Aruba mgmt port descriptions set.
- 09/09 — Ring design finalized, port names corrected, Aruba STP verified.
- 09/10–09/16 — silence (no notes).
- 09/17 — Brain upgrade tenkaichi-budokai completed, timeout fixed.
- 09/18 — Workflow directives added, config layering applied.

Design documents: docs/garage_cluster.md (ring design with RSTP), docs/network-topology.md (host table, port mapping, ASCII diagram).

VyOS pre-freeze configs: logs/vyos-freeze/.

AdGuard query log export (STB investigation, not this week): logs/stb-192.168.33.101-adguard-2026-09-14.zip.

PromQL queries attempted but not returned: Prometheus proxy at 10.22.0.150:9090 unreachable.

AGENTS.md: directives 11–20 as of 09/18.

The week's evidence is thin — two activity notes and the interview — because everything else is silent. The interview confirms what the notes document; the notes confirm what the interview describes. That is the best we have for a week where the cluster is down and the brain is the only thing moving.
