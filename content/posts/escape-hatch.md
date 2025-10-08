+++
title = "Escape Hatches for Rollups: Why Users Need Forced Exit Tools"
description = "sharing my thoughts on the importance of escape hatches for rollups and how they can empower users."
date = 2025-10-07  
updated = 2025-10-07
# draft = true   

[taxonomies]
categories = ["blockchain","rollups"]
tags =["rollups","l2","ethereum", "forced exits"]

[extra]
lang = "en"
toc = true  # table of contents
comment = false  # enable comments
copy = true  # copy code button
outdate_alert = true  # show outdated alert
outdate_alert_days = 120  # days before showing alert
math = false  # enable math rendering
mermaid = false  # enable mermaid diagrams
featured = false  # feature on homepage
reaction = false  # enable reactions
image = "/escape-hatch/escape-hatch-cover.png"
+++

## Opening Hook

There was a recent back-and-forth on CT about Base sequencers and whether they can really control user funds. The key point: sequencers can’t just steal your ETH or stop withdrawals. Vitalik even broke this down in a tweet:

<div style="display: flex; justify-content: center;">
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Base is doing things the right way: an L2 on top of Ethereum, that uses its centralized features to provide stronger UX features, while still being tied into Ethereum&#39;s decentralized base layer for security.<br><br>Base does not have custody over your funds, they cannot steal funds or… <a href="https://t.co/0EMdThg4gU">https://t.co/0EMdThg4gU</a></p>&mdash; vitalik.eth (@VitalikButerin) <a href="https://twitter.com/VitalikButerin/status/1970258498996048247?ref_src=twsrc%5Etfw">September 22, 2025</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>

He explains that rollups have built-in protections. One of the big ones is _forced transaction execution,_ if a sequencer censors you, you should still be able to push a withdrawal through and get your funds out without their cooperation.

As a developer, I can script my way through this. But then I wondered: how does a normal user, who’s never touched calldata, do the same? Right now, there’s no straightforward way.

## Why This Matters

Rollups inherit Ethereum’s security _only if users can always exit back to L1_. In good times, sequencers behave and withdrawals go through fine. But in emergencies — censorship, downtime, governance capture, that safety net matters. Forced execution is the “emergency exit door.” It ensures your funds are protected by Ethereum’s economic security, not the goodwill of a single sequencer.

This is exactly what makes rollups powerful: the combination of Ethereum’s L1 guarantees with the property that you can always force your way out.

## Rollup Stages (L2Beat)

L2Beat frames rollup maturity using “stages”:

- **Stage 0** → fully trusted setup, operators can block or fake messages.
- **Stage 1** → users are safe as long as a security council (≥75%) isn’t compromised, and upgrades must allow exits.
- **Stage 2** → fully permissionless fraud/validity proofs, minimal reliance on any council.

For the purposes of this blog (and the project I’m working on), the minimum bar we care about is **Stage 1**. That’s where forced exits are guaranteed by design.

Not every rollup is there yet, for example, zkSync (which I’ll dive into later) doesn’t currently meet Stage 1. But the important part is: the escape hatches exist at the protocol level, altho they can be controlled by the governance in this case.

## What I Found Digging Deeper

I dove into three prominent rollup stacks — **Arbitrum Nitro**, **OP / Optimism (Bedrock)**, and **zkSync Era,** to understand exactly how forced execution or forced withdrawal works (or is supposed to work). The core logic differs per stack, but the goal is always: “let the user bypass a censoring sequencer and safely exit.”

I’ll walk through each, then explain the generalized flow. After that, we’ll see _where_ the tools break for regular users, that’s where our missing link lies.

### Arbitrum Nitro

- Arbitrum maintains a **delayed inbox** on L1. Users can enqueue transactions there.
- If the sequencer ignores or fails to include such a message within a threshold (currently ~24 hours on Arbitrum One) anyone can call `forceInclusion` on the sequencer inbox contract on L1 to force that message into the rollup’s sequence.
- This ensures the transaction _eventually_ is processed, even if the sequencer is censoring.

### Optimism / OP Stack (Bedrock)

- Optimism defines **deposit transactions** via its `OptimismPortal` contract on L1. Users can post an L2 transaction by calling `depositTransaction` on L1, effectively enqueueing it into an L2 message queue.
- The rollup’s derivation rules require that these L1-originated messages will be included in L2, even if the sequencer is down. Under extended downtime, after ~12 hours, the protocol forces inclusion of those deposits.
- In short: if the sequencer is offline or censoring, your L1‐submitted transaction will eventually be processed because of protocol rules.

Optimism’s “forced transaction” mechanism is documented in their docs.

### zkSync Era

- zkSync has an L1 → L2 **priority queue / mailbox**. Users or contracts can call a function (e.g. `requestL2Transaction`) in the L1 mailbox to submit a request (withdrawal or action) destined for L2.
- The sequencer is required by protocol rules to process the L1-originated messages in order, and not arbitrarily reorder or skip them.
- If the sequencer completely halts or refuses to act, in theory a user (or third party) can initiate a **trustless exit** by submitting a zk-proof to the L1 contract capturing their funds and state after some time. The idea is that because zk proofs validate state transitions, you can bypass the sequencer. (However, in practice zkSync is not yet fully permissionless in who can submit proofs.) Because zkSync still restricts who can publish state roots, it does **not** fully satisfy Stage 1 under L2Beat’s framework.

### The Generic Forced Withdrawal Flow

Here’s a distilled “recipe” of how forced withdrawals are supposed to work across rollups:

1. **Craft the withdrawal transaction** on L2 (or a call to a contract that releases your funds). For native tokens, this usually means calling the standard bridge contract’s method to initiate the exit.
2. **Submit via L1** — call the corresponding rollup’s L1 contract (e.g. Optimism’s `OptimismPortal`, Arbitrum’s inbox, zkSync’s mailbox) containing your withdrawal intent.
3. **Inclusion & execution on L2** — eventually that message is included in an L2 block (either by sequencer or forced inclusion). The withdrawal executes on L2.
4. **State posting to L1** — the rollup posts the resulting state (or output root) on L1 (batch), so L1 “sees” the new L2 state.
5. **Construct inclusion / withdrawal proof** (for optimistic rollups) — use proof machinery to show that your withdrawal transaction was indeed part of the posted state (via Merkle/inclusion proofs).
6. **Finalization / claim** — after any challenge window passes (for optimistic rollups), call a function on L1 (e.g. `finalizeWithdrawal`) to actually move funds to L1 (or send tokens).

Depending on rollup type:

- **Optimistic** → you need to wait the challenge period, and potentially submit fraud proofs / proof-of-inclusion.
- **zkRollups** → because of validity proofs, invalid states are rejected outright; once your state is accepted, finalization is more straightforward.

In practice, some steps require access to archive data, correct proofs, or state roots, so not trivial for a casual user.

## What’s Missing Today?

If you’ve read through the process above, you can already see the problem: it’s too complex for anyone outside of a developer circle. The contracts, inboxes, proofs, challenge periods, they’re technically there, but hidden behind low-level calls and infra requirements.

A few tools and experiments have tried to bring parts of this vision to life, but none yet fully unlocks escape hatches for normal users.

- **Superbridge** supports _forced withdrawals_ as an escape hatch across rollups. But it’s tied into a bridging UX and isn’t a standalone, backend-agnostic tool.
- **Arbitrum Connect by WakeUp Labs** is a front-end interface to force transaction inclusion during sequencer downtime on Arbitrum. This is closer to what I’m after, but it's token specific.
- **dYdX escape hatch** (built by L2Beat with Starknet), but that’s not general-purpose.

## My Idea: An Open-Source Escape Hatch

This led me to sketch an idea: a **standard, open-source platform** that makes forced transaction execution practical. Think of it as a universal “emergency exit app” for rollups.

<div style="display: flex; justify-content: center;">
  <img src="/escape-hatch/escape-hatch-cover.png" alt="Escape hatch cover" style="max-width: 60%; height: auto; border-radius: 8px;" />
</div>

Here’s how I imagine it working:

- **Frontend** → lightweight HTML/JS app, hosted on IPFS, resolved via ENS for censorship-resistant access.
- **Backend logic** → directly integrates with rollup contracts and libraries, minimal third-party dependencies.
- **Node access** → uses rollup nodes to fetch proofs/state; in failure scenarios, users could run their own node to complete the process.
- **Self-reliance** → not dependent on any centralized service. Users could even spin up their own local copy.
- **Emergency mode** → fallback CLI, works standalone with your own node if everything else is down.
- **Open source** → MIT-licensed, transparent, auditable.

To start, I’d scope it down to **ETH and ERC-20 forced exits**. Those are relatively simple and the most immediately useful. More complex cases (like exiting from a DeFi protocol position) will take time, since those require multi-step contract interactions and more complex proofs.

## What’s next?

The purpose of this first blog is to frame the problem: escape hatches exist at the protocol level, but not at the user level. If we don’t solve that, we don’t really get the full promise of rollups.

This will be the first part of a series:

- **Part 1** (this post): the problem + research.
- **Part 2**: a sketch/spec of the platform.
- **Part 3**: an MVP implementation (frontend + CLI).

I’d love for this to be collaborative. If you’re working on rollups, infra, or care about Ethereum’s trust model, I’d really appreciate thoughts, feedback, and contributions. my [twitter](https://twitter.com/0xdhruva) DMs are open.

## Acknowledgments

Thanks to [Rampey](https://x.com/0xRampey), [Dhaiwat](https://x.com/dhaiwat10) for their feedback and insights on this post.
