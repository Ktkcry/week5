# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **NNNN**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

At first from the s2 tunnel table we know that , the entry `dst_id = 3` had handle `0x2` and sent traffic out of port 3 (to h3). Then I redirected it back towards s1 with:

```
sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"
```

s1 already sends exit-3 traffic to s2 (port 2), so the two switches now pass the packet back
and forth. I then sent one tunnelled packet:

    h1 python3 send.py 10.63.0.3 "loop" --tunnel 3

I counted the lines "Processing packet received" in the switch logs:

| Time | logs/s1.txt | logs/s2.txt |
|---|---|---|
| Straight after sending | 1253 | 1253 |
| 5 seconds later | 609 | 608 |

No it does not stop the loop packet because the  tunnelled packet never uses the IPv4 path. In the ingress `apply` block it goes to
`myTunnel_exact`, whose action `myTunnel_forward` only sets the output port. The only code that changes the TTL is `ipv4_forward`, which tunnelled packets do not run. 

## 2. Design: the hop limit


**Field.** I add `bit<8> hop_limit` to `myTunnel_t`, after `dst_id` (header goes from 4 to
5 bytes). It means "how many more switches this packet may cross". 8 bits is the same
width as IPv4's TTL and covers 0–255, far more than this 2-switch network needs, so it adds
only one byte per packet. I put it last so `proto_id` and `dst_id` keep their original
offsets, which keeps the parser's `select` on `proto_id` unchanged and should make the
mixed rollout in C5 easier to reason about.

**Starting value.** `send.py` sends 8 by default, with `--hops N` to override. The longest
normal path crosses 2 switches, so 8 leaves headroom for longer paths, while a loop still
ends after only 8 passes.

**Decrement and zero.** In the ingress tunnel branch, each switch checks the field before
changing it: if `hop_limit == 0` it calls `drop()`; otherwise it subtracts 1 and applies
`myTunnel_exact`. The check must come first, because P4 arithmetic wraps (0 - 1 = 255), so
a decrement alone would never stop a loop. Doing it in ingress means a doomed packet is
dropped before it reaches a queue, and it is one comparison and one subtraction per
switch, which fits the single-pass, no-loop pipeline. The logic is in the program (compile
time), not in table entries. Plain IPv4 traffic is untouched.

**Expected loop result (for C4).** A packet starting at N is forwarded by N switches and
dropped by the next, so the C1 loop should show N + 1 packet-processing events in total
(9 for N = 8).

**What else changes so every part agrees.**
- `tunnel.p4`: add the field to `myTunnel_t` (the parser's `extract` and the deparser's
  `emit` then include it automatically; the parser's `select` and the deparser's `emit`
  order do not change).
- `send.py`: add the field to its tunnel header definition, set from `--hops`.
- `receive.py`: same header layout, and print the arriving hop limit.

## 3. Implementation

**tunnel.p4.** I added `bit<8> hop_limit` to the end of `myTunnel_t` and changed the tunnel
branch of the ingress `apply` block: if `hop_limit == 0` the packet is dropped; otherwise
the field is decremented by 1 and `myTunnel_exact` is applied. Plain IPv4 handling is
unchanged.

**send.py and receive.py.** Both scapy `MyTunnel` classes got a matching
`ByteField("hop_limit", 8)`. `send.py` has a new `--hops H` option (default 8, rejected
outside 0-255) that sets the starting value on tunnelled packets only. `receive.py` now
prints the arriving value, e.g. `tunnelled, dst_id=2, hop_limit=6`.

## 4. Evidence: it works

## 4. Evidence: it works

**Delivery (delivery.txt).** Tunnelled packets arrived at h2 and h3 with `hop_limit=6` (start 8,
minus 1 at each of the two switches) and TTL 64, since the tunnel path does not touch the TTL.
`h1 ping -c 2 h3` still worked with TTL 62, so plain IPv4 is unchanged.

**Loop now ends (loop-after.txt).** On a fresh network with s2's exit-3 entry redirected to
port 1, one packet with hop limit 8 gave counts of 1/1 (s1/s2) before and 6/5 after, so the
loop packet was processed 5 times on s1 and 4 on s2: 9 in total. The counts did not change
later, so the loop stopped. (Section 1: over a thousand events, never ending.)

**Match with design.** Yes: the design predicts N + 1 = 9 events for N = 8. Eight visits
forward the packet and lower the hop limit to 0; on the ninth, s1 sees 0 and drops it.

## 5. Evaluation

**a) Mixed rollout (mixed.txt).** *Prediction:* s1 (new) lowers the hop limit and forwards by
`dst_id`. Old s2 does not know the hop byte, so it misreads the packet, but it should still
forward it because `dst_id` is in the same place. I expected the packet to arrive damaged, and the
loop to take twice as long to stop (17 events instead of 9), because only s1 counts hops.

*Observed:* the packet reached h2 with hop limit 7, so s2 did not lower it. s2 read 38 bytes
instead of the new program's 39, so it took the hop byte as the start of the IPv4 header. It then
overwrote the IP protocol byte (`0x11` became `0x0d`), which breaks the packet.

**b) Safe rollout.** *Field order:* putting `hop_limit` last keeps `dst_id` at its old offset, so
old switches still forward, but they misparse IPv4 and corrupt it, so order alone is not safe.
*Version field:* old switches never check it, so it only helps once all switches read it.
*New EtherType:* old switches do not recognise it and drop the packet, which is safe (fail-closed,
no corruption). I recommend the new EtherType in stages: (1) upgrade every switch to parse both
formats, (2) switch senders to the new format, (3) remove the old one.

**c) Data plane vs control plane.** The hop limit stops any loop that exists now, whatever the
cause, with no controller, but it only limits the damage, does not say which entry is wrong, and
covers only tunnelled packets. `check_paths()` can find the bad
entry before traffic is sent, but cannot see what switches actually do or packets already in
flight. Real networks use both: one prevents, the other contains.

**d) Line-rate cost.** One extra byte per packet, one extra field to parse and emit, and one
comparison plus one subtraction per switch. Latency stays constant, but the `if` likely needs an
extra table/stage (of about 10-20) and 8 bits of header space; if it does not fit, the compiler
rejects it.

## AI use

Use claude in learning and assisting in generate code .

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
