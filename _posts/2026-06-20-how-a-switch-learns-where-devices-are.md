---
layout: post
title: "How a Switch Learns Where Devices Are"
date: 2026-06-20
description: "An introduction to MAC addresses, MAC address tables, learning, flooding, and forwarding."
tags:
  - networking
  - switching
  - layer-2
---

---

## Introduction

Imagine you’ve just plugged three laptops into a switch.

Host A wants to send a frame to Host B.

The switch has just been powered on and knows nothing about the devices connected to it.

How does it figure out where Host B is located?

The answer lies in MAC addresses and something called a MAC address table.

---

## Switches

A switch is a Layer 2 (Data Link layer) device that is used to transfer data/facilitate communication between hosts *within* a network.

Switches use and maintain a MAC address table to keep track of which devices are reachable through which ports.

---

## MAC Addresses

A MAC address is a Layer 2 identifier associated with a network interface. It is typically represented as six pairs of hexadecimal digits, for example:

```
AA:BB:CC:DD:EE:FF
```

## MAC Address Table 

Switches use and maintain a MAC address table.

Each host is connected to a particular switch port.

The switch stores these relationships in a MAC address table, which maps MAC addresses to switch ports.


```
AA:AA -> Port 1
BB:BB -> Port 2
```

Switches build and use their MAC address tables through three key behaviours:

- Learning
- Flooding
- Forwarding

---

## Learning

When a switch first powers on, its MAC address table is empty.

Suppose Host A sends a frame into Port 1.

The switch examines the source MAC address of the incoming frame and records that MAC address AA:AA is reachable through Port 1.

This information is added to the MAC address table so the switch can make forwarding decisions in the future.

This process is known as learning, where a frame arrives, the switch learns the source MAC address of the frame, and updates the MAC address table with a mapping of `<switch port>:<MAC address>`.

Over time, the switch gradually builds a map of where devices are connected.

### The Catch

Learning only records where source MAC addresses are located.

After receiving the first frame from Host A, the switch knows where AA:AA resides.

However, it still has no idea where BB:BB resides.

This is where flooding comes in.

## Flooding

If the switch has just been powered on, then the destination MAC address may be unknown to the switch.

In this case, the switch sends the frame out on all switch ports, except for the receiving port.

This process is known as flooding.

One reason a switch floods traffic is when it receives a unicast frame (a frame intended for a single, specific device) destined for a MAC address that is not yet present in its MAC address table. 

This behaviour is known as unknown unicast flooding.

## Forwarding

Once the `<switch port>:<MAC address>` mapping is learned and populated in the MAC address table, the switch uses the MAC address table to deliver subsequent frames to the appropriate switch port only.

## Putting It All Together

We’ve now seen the three key behaviours a switch uses:

- Learning
- Flooding
- Forwarding

We will walk through a complete example and see how they work together.

## Example

```
Host A           Host B
AA:AA            BB:BB
  |                |
Port 1          Port 2
   \            /
     \        /
      Switch
         |
      Port 3
         |
      Host C
      CC:CC
```

**Step 1:**

Host A wants to send a frame to Host B.

But the switch has an empty MAC address table.

```
MAC Address Table

(empty)
```

**Step 2:**

The frame arrives on Port 1 with a source MAC address of AA:AA.

The switch immediately records the mapping in its MAC address table:

```
MAC Address Table

AA:AA -> Port 1
```

Even though the switch doesn’t know just yet where Host B is located, it has now learned where Host A resides.

**Step 3:**

But the switch doesn’t know which port BB:BB is connected to.

So the switch sends the frame out on all the switch ports (except for the receiving port).

```
Host A           Host B
AA:AA            BB:BB
  |                |
Port 1          Port 2
   \            /
     \        /
      Switch
         |
      Port 3
         |
      Host C
      CC:CC

Frame arrives for BB:BB

Flood:
→ Port 2
→ Port 3
```

**Step 4:**

Host B receives the frame and responds.

When Host B responds, the frame enters the switch through Port 2. The switch examines the source MAC address of this frame (BB:BB) and learns that BB:BB is reachable through Port 2.

The switch then updates its MAC address table once again:

```
MAC Address Table

AA:AA -> Port 1 
BB:BB -> Port 2 
```

**Step 5:**

Now that the MAC address table contains mappings for AA:AA and BB:BB, future traffic between Host A and Host B is forwarded directly to the appropriate ports, with no further learning or flooding required.

## Why This Matters For Kubernetes Engineers

Modern Kubernetes clusters may use virtual switches, bridges, overlay networks, VXLAN, and cloud networking constructs, but the underlying principles remain the same.

Understanding how switches learn MAC addresses provides a foundation for understanding more advanced topics such as VLANs, overlay networking, OVN-Kubernetes, and cloud networking.

## Follow-On

In the next article, we’ll look at a related question: if hosts communicate using MAC addresses at Layer 2, why do we also need IP addresses? The answer lies in the Address Resolution Protocol (ARP).