# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **2616**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

No, the IPv4 TTL does not stop a loop in this program. The ipv4_forward action decreases the TTL by 1, but the program does not check whether the TTL reaches 0. Therefore, it does not automatically drop the packet.

Also, hdr.ipv4.isValid() only checks if the IPv4 header is valid. It does not check the TTL value.

For tunnel packets, the program uses myTunnel_exact instead of ipv4_forward, so the IPv4 TTL is not decreased. This is why we need to add a hop limit to the tunnel header.

In `evidence/loop-before.txt`, exit 3 on s2 was changed using handle 2. The recorded counts were 261/260 and then 25/22 for s1/s2. One possible explanation is that the log files were rotated or recreated, so the later count covered a new file rather than a counter overflowing and restarting at zero. The recorded output alone does not confirm the cause or prove that the loop continued.

## 2. Design: the hop limit

I added a new field called hop_limit to the tunnel header.

1. Field size and position

I chose 8 bits (1 byte) for the hop limit. It can store values from 0 to 255, which is enough for our small network. I placed it after dst_id to keep the original fields in the same order.

2. Starting value

I chose 4 as the starting value. Our normal path uses two switches, so 4 is enough for the packet to reach its destination. It also limits the number of forwarding steps if a loop happens.

3. Where the value is decreased

The hop limit is decreased by 1 in the ingress pipeline before the tunnel forwarding table is applied.

4. What happens when it reaches zero

When a switch receives a tunnel packet with a hop limit of 0, it drops the packet. This prevents the packet from looping forever between switches.

## 3. Implementation

In `tunnel.p4`, the parser and deparser use the extended tunnel header. The ingress checks the received hop limit before decrementing it: zero is dropped; otherwise the value is reduced and `myTunnel_exact` selects the output port. Thus a packet forwarded with a value of zero is dropped at the next upgraded switch. The IPv4 path remains separate.

Both `send.py` and `receive.py` define the same extra one-byte field after `dst_id`. The sender accepts `--hops` (default 4), and the receiver prints the received value. All three must agree on the wire format.

## 4. Evidence: it works

In `evidence/delivery.txt`, packets sent with hop limit 4 reached h2 and h3 with limit 2 and IPv4 TTL 64. The plain ping from h1 to h3 received both replies, with 0% loss and TTL 62.

In `evidence/loop-after.txt`, after a second loop packet was sent with limit 4, the recorded counts increased from 684 to 687 on s1 and from 680 to 682 on s2, then remained 687 and 682 after five seconds. The differences are three receptions at s1 and two at s2, consistent with four forwarding operations followed by a fifth reception at s1 where a zero limit is dropped. These are aggregate switch counts rather than packet-specific traces, so unrelated traffic cannot be ruled out from the counts alone.

## 5. Evaluation

### Mixed rollout

A prediction made before the test is not recorded in the available report or evidence. From the two header definitions, the expected compatibility issue is that the old switch reads the new hop-limit byte as the first byte of IPv4, shifting its interpretation of the IPv4 header. It still sees the same first four tunnel bytes and can use the destination ID.

In `assignment/mixed.txt`, the capture at h2 shows EtherType `1212`, then `0800 0002 03 45...`: protocol 0x0800, destination ID 2, hop limit 3, and the actual IPv4 first byte 0x45. The old s2 log shows 18 bytes parsed after its four-byte tunnel header, followed by IPv4 extraction, and a lookup for destination ID 2. The receiver reports the packet at h2 with hop limit 3. s1 reduced 4 to 3; the old s2 did not reduce it. Receiving the packet does not mean s2 correctly interpreted the IPv4 header. The capture and log support the byte-offset explanation; they do not by themselves establish the validity of every IPv4 field or checksum.

For the mixed loop, the recorded aggregate counts were 694 and 688 at both observations five seconds apart. This is consistent with the loop having stopped, but the record has no immediately preceding baseline or packet-specific drop trace, so the number of passes and the exact drop cannot be established from it. The evidence names `tunnel.p4_old` in the s2 log; the startup command is not included, so the precise startup configuration is not independently shown.

### Rollout options

One option is a new EtherType: first deploy switches that understand both formats, then move senders to the new EtherType, and retire the old format after traffic has migrated. This distinguishes formats before parsing but needs support for both during migration. A version field can also select a parser, but only if it is at a position understood by both versions; adding one to this existing unversioned header would itself break old parsers. Either approach needs a compatibility phase, and traffic in the new format must only traverse switches able to parse it or use a translator at the boundary.

### Data plane and control plane

A hop limit bounds the lifetime of an individual packet even if a transient or unexpected forwarding loop occurs after configuration. It does not identify the faulty rule and may drop a legitimate packet if the limit is too small. A control-plane path check can inspect the rules and topology to detect a loop before traffic is sent and help locate the configuration problem, but its conclusion depends on the accuracy and freshness of its model and cannot cover every transient state. Using both combines prevention and diagnosis with a per-packet bound.

### Hardware cost

Each tunnelled packet carries one extra byte, consuming link bandwidth and potentially affecting MTU or alignment. A switch must extract and emit the field and perform a zero comparison and decrement in the pipeline, using parser, header-vector and processing resources. The actual area, timing and throughput impact depends on the target hardware and compiler; this software experiment does not measure line-rate cost.

## AI use

AI Tool: Chat GPT

Function: It give me some instruction for the assignment, e.g. code. And help me to understand why need some code and told me some codes functions

Chat Link: [Lab 5 Assignment with Chat GPT](https://chatgpt.com/share/6af419f4-5030-83ec-8ce2-9302fa5e841d)

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The original loop setup and two inconsistent count samples |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | Expected path; the actual record is `../mixed.txt` |
