# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **4939**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

<!-- How did you create the tunnel loop (which entry, on which switch, which commands)?
     What did the switch logs show over time, with numbers from loop-before.txt?
     Then look at the IPv4 path in tunnel.p4: it decrements the TTL. Does that stop an IPv4
     loop in this program? Check the code, and explain what happens when TTL reaches 0. -->

I created a forwarding loop by modifying the S2 forwarding-table entry for destination ID 3. Normally, this entry forwards packets through port 3 to H3. I changed it to port 1, which connects S2 back to S1.

The commands used were:
sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"
sh ./p4cli s2 "table_dump MyIngress.myTunnel_exact"

The table dump confirmed that destination ID 3 was forwarded through port 1 instead of port 3.

This caused packets destined for H3 to circulate between S1 and S2 instead of reaching H3. The forwarding-table configuration is shown in evidence/loop-before.txt.

The IPv4 forwarding path in tunnel.p4 decrements the IPv4 TTL when forwarding IPv4 packets. However, the tunnel forwarding path uses a separate action and does not decrement the IPv4 TTL. Therefore, the IPv4 TTL alone does not protect this tunnel forwarding loop.

The tunnelled packet can continue circulating because its IPv4 TTL is not modified by the tunnel forwarding action. This demonstrates the need for a separate hop-limit mechanism for tunnelled packets.

## 2. Design: the hop limit

<!-- Your header change and your pipeline change, and the reason for each choice:
     - the field's size, and where in the header you put it (and why there);
     - the starting value send.py uses, and how you chose it;
     - where it's decremented, and what happens when it runs out;
     - what else had to change for every part of the path to agree (parser, deparser, hosts). -->

I added an 8-bit hop_limit field to the custom tunnel header, after the existing proto_id and dst_id fields.

header myTunnel_t {
    bit<16> proto_id;
    bit<16> dst_id;
    bit<8> hop_limit;
}

An 8-bit field was chosen because it supports values from 0 to 255, which is sufficient for limiting the number of forwarding operations in this network. The field was placed after dst_id to extend the existing header without changing the positions of the original fields.

The sender initializes the hop limit to 16 by default. This value provides enough forwarding allowance for the intended network while ensuring that packets cannot circulate indefinitely.

The forwarding action checks whether the hop limit is zero before forwarding. If it is zero, the packet is dropped using mark_to_drop(). Otherwise, the hop limit is decremented by one before forwarding.

The check is performed before decrementing to prevent the unsigned 8-bit value from underflowing from zero to 255.

## 3. Implementation

<!-- Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows the code itself: here, explain anything a reviewer might not see from the diff alone. -->

The main changes were made in tunnel.p4, send.py and receive.py.

In tunnel.p4, I added the hop_limit field to the tunnel header and updated the parser and deparser. I also modified the tunnel forwarding action to check and decrement the hop limit. Packets with a hop limit of zero are dropped instead of being forwarded.

In send.py, I added the --hops argument to allow the initial hop limit to be configured. For example:
h1 python3 send.py 10.39.0.3 "hello h3" --tunnel 3 --hops 16

In receive.py, I updated the receiver to display the destination ID and hop-limit value of received tunnelled packets. This allows the hop-limit behaviour to be verified through the receiver logs.

One important implementation detail is that the hop limit is decremented at every switch that successfully forwards a tunnelled packet. This ensures that the mechanism works independently of the destination and can prevent forwarding loops.

## 4. Evidence: it works

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match your design? (loop-after.txt) -->

TUNNELED PACKET DELIVERY
I sent tunnelled packets from H1 to H2 and H3 using an initial hop limit of 16.

The receiver logs showed:
Received (tunnelled, dst_id=2, hop_limit=14)
Received (tunnelled, dst_id=3, hop_limit=14)

Both packets arrived successfully with a hop limit of 14. This is because each packet passed through S1 and S2, and both switches decremented the hop limit once.

NORMAL IPV4 CONNECTIVITY
I also tested normal IPv4 connectivity using:
h1 ping -c 2 h3

The result was:
2 packets transmitted, 2 received, 0% packet loss

This confirms that normal IPv4 forwarding continued to work after implementing the tunnel hop limit.

FORWARDING LOOP PROTECTION
I repeated the forwarding-loop experiment with both switches running the new tunnel.p4 program. The packet was sent with an initial hop limit of 16.

The logs showed the following results:
Packet processing events: S1: 9, S2: 8
Successful hop-limit decrements: S1: 8,	S2: 8

There were 16 successful forwarding operations in total. After these operations, the hop limit reached zero. S1 received the packet for the 17th processing event and dropped it using mark_to_drop().

Therefore, the forwarding loop was successfully terminated. The result matches the design because the packet was dropped when its hop limit reached zero.

The results are recorded in evidence/loop-after.txt.

## 5. Evaluation

<!-- a) The mixed rollout (mixed.txt): what did you predict before running s1 with your new program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop? Explain every difference from your prediction, byte by byte if you need to.
     b) Propose a safe way to roll out a header change across a network of switches that can't all be upgraded at once. Compare at least two options (for example a version field, a new EtherType, or ordering fields so old switches still parse them).
     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's check_paths()): what can each catch that the other can't? Why do real networks use both?
     d) What would this change cost on a hardware switch running at line rate? -->

a) 
Before running the experiment, I expected that using different tunnel-header formats between S1 and S2 could cause compatibility problems. S1 was running the new tunnel.p4 program, while S2 was running the old tunnel_v1.p4 program.

The new tunnel header contains an additional 8-bit hop_limit field, making it five bytes long instead of four bytes.

The packet layout is:
proto_id = 2 bytes
dst_id = 2 bytes
hop_limit = 1 byte
IPv4 header starts = After 5 bytes

However, the old S2 program expects the IPv4 header to start immediately after the original four-byte tunnel header.

Consequently, S2 starts parsing the IPv4 header one byte too early. It interprets the hop-limit byte as the first byte of the IPv4 header. Although S2 can still extract the destination ID and match its forwarding-table entry, it misinterprets the IPv4 packet.

The H2 packet capture showed zero packets received.

For the forwarding-loop experiment, S2 continued forwarding packets without decrementing the hop limit because its old program did not contain the new hop-limit logic. S1 continued decrementing the hop limit whenever the packet returned.

The logs showed that S1 processed 17 packets and S2 processed 16 packets. S1 performed 16 decrements before dropping the packet when the hop limit reached zero.

This demonstrates that changing a packet-header format without upgrading all switches can cause packet-parsing and compatibility problems.

b)
Two possible approaches can be used to introduce a new tunnel-header format while allowing switches to be upgraded gradually.

Approach 1: Add a version field
A version field can be added to the tunnel header to identify its format. Switches can inspect this field and select the appropriate parser and forwarding logic.
- Advantages: Different header versions can coexist, allowing gradual upgrades.
- Disadvantages: Requires additional header space and more parser logic.

Approach 2: Use a new EtherType
A different EtherType can identify packets using the new tunnel-header format. New switches can distinguish between old and new packet formats before parsing the tunnel header.
- Advantages: Clearly separates different packet formats and avoids interpreting a new header as an old one.
- Disadvantages: Requires changes to packet generation and switch configuration.

Both approaches can improve compatibility during network upgrades. A version field is useful when multiple header versions need to coexist, while a new EtherType provides a clear distinction between different packet formats.

c)
The control-plane check_paths() function from Week 3 checks the forwarding paths configured in the network. It can detect loops in the known network topology before forwarding rules are installed.

However, it cannot guarantee that packets will never encounter a loop caused by incorrect or unexpected forwarding-table entries.

The data-plane hop-limit mechanism protects packets while they are being forwarded. Each switch decrements the hop limit, and packets are dropped when the value reaches zero.

The two mechanisms provide different protections:
check_paths() = Detects loops in configured forwarding paths before forwarding.
Hop limit = Stops packets from circulating indefinitely during actual forwarding.

Real networks use both mechanisms because preventing incorrect forwarding configurations is important, but unexpected loops can still occur. The control plane helps prevent loops from being configured, while the data plane provides protection when packets actually encounter them.

d)
Implementing the hop limit introduces additional processing requirements for a hardware switch.

The additional costs include:
1. Header storage: The tunnel header increases by one byte because of the new 8-bit field.
2. Parser resources: The parser must extract the additional field.
3. Comparison operation: The switch must check whether the hop limit equals zero.
4. Arithmetic operation: The switch must decrement the hop limit for successfully forwarded packets.
5. Conditional dropping: The switch must decide whether to forward or drop the packet based on the hop-limit value.
These operations consume hardware resources, including parser capacity, header storage, arithmetic resources and match-action pipeline resources.

Although these operations are relatively simple, they must be executed within the switch's packet-processing pipeline to maintain line-rate forwarding. Therefore, the additional processing cost should be considered when implementing this mechanism on high-speed hardware switches.

## AI use

<!-- This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you
     used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format;
     the first draft of the hop-limit check came from Copilot, which I then changed and tested".
     Be ready to explain any part of your work yourself. -->

ChatGPT was used to explain P4 concepts, help debug implementation issues and interpret BMv2 logs. The implementation and testing were performed in the Mininet environment.

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | S2 forwarding-table modification that creates the loop. growing |
| `evidence/delivery.txt` | Successful tunnelled delivery to H2 and H3, received hop-limit values, and successful IPv4 ping. |
| `evidence/loop-after.txt` | Packet-processing counts, hop-limit decrements and the final drop event. |
| `evidence/mixed.txt` | Mixed-version rollout results, including failed packet delivery and loop behaviour. |
