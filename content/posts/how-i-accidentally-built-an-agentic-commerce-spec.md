+++
title = "How I accidentally built the first agentic commerce spec, and almost missed why"
description = "Building pipegate: a payment middleware spec for machine-to-machine transactions, missing the 402 status code, and learning about market timing through x402."
date = 2026-07-01
updated = 2026-07-01
# draft = true  # Uncomment to keep as draft

[taxonomies]
categories = ["finance", "blockchain", "payments"] 
tags = ["agentic", "payments", "x402", "pipegate"]

[extra]

lang = "en"
toc = true  # table of contents
comment = false  # enable comments
copy = true  # copy code button
math = false  # enable math rendering
mermaid = false  # enable mermaid diagrams
featured = true  # feature on homepage
reaction = false  # enable reactions
image = "/agentic-commerce-spec/cover.png"
+++

## ![agentic-commerce-spec-cover](/agentic-commerce-spec/cover.png)

it was october 2024. i came across a thread from georgios talking about how broken API access was for developers juggling 10 different API keys and credit systems, and for providers who couldn't easily accept global payments without losing 3-4% to fees or losing customers who didn't want a full subscription. the core idea was simple: pay per use, no friction, no keys.

somewhere in that thread, someone had already built a rust side component — a wrapper around an axiom server that gated incoming requests based on payment channel config. it was half-cooked. no clear spec, no client-side story, no onchain detail. but the shape of the idea was right.

i wanted to finish it.

## context: why this felt urgent

at this point, i wasn't thinking about agents at all. this was purely a developer UX problem. API keys are annoying. subscriptions are overkill for small usage. providers leave money on the table because their payment stack doesn't reach everyone.

i was also learning rust at the time, coming off my last role, i'd been wanting to contribute more to open source, and rust made sense given how much of the infra i cared about (reth, helios, alloy) was written in it. the crypto devtool ecosystem in rust was genuinely strong, and the memory/thread control was interesting. steep learning curve, but worth it.

the timing felt right to try something real with it.

## bangkok: the all-nighter spec

EthGlobal Bangkok 2024, happening during Devcon Thailand — seemed like the right place to test this. i convinced my teammate kushagra to build with me on it, which itself took some work. even i was finding it hard to articulate _why_ this was a big deal at the time.

we set ourselves a tight scope for the hackathon: an end-to-end flow. a typescript client wrapping axios with payment channel logic built in, a `ChannelFactory` and `PaymentChannel` smart contract deployed on Base, and a rust server middleware that verified incoming requests and routed them accordingly. the consumer would deposit USDC into a payment channel upfront and use it across concurrent API calls.

we pulled an all-nighter to get a working POC together. demoed it to the Base and Circle teams. they liked the vision. we won the Circle bounty.

getting back from bangkok, i knew this needed a proper version.

## building the real spec

the post-hackathon phase is where the actual thinking happened.

i went deep on tokio internals to figure out the right abstraction layer for the server side. on the client side i studied http libs like axios carefully. i also made a call early: typescript and rust as the two supported environments. most end user apps were on some TS framework — Next, Vue, etc. rust was the right choice for a different reason though: it compiles to WASM, which means one implementation that ports to almost anything. i built in `wasm32` target support from the start even though it added real effort, because i knew it was the right call.

the flow i settled on had three steps for developers to start accepting payments. the consumer creates a payment channel onchain, and it's stored as config. on every subsequent request the client middleware auto-adds the required headers. the server middleware extracts and verifies them, routes the request, auto-deducts the balance, and returns the updated state as a response header. near end of channel lifecycle, the provider extracts funds from the contract using the last agreed state + consumer signature. remaining balance refunds automatically.

i turned this into an excalidraw flow and shared it on twitter

<div style="display: flex; justify-content: center;">
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">i hate api keys,<br><br>and as a web3 dev, i tried to solve this with one-way payment channels + stablecoins<br><br>one wallet, zero api keys, simple and fast API monetisation with USDC on <a href="https://x.com/base?ref_src=twsrc%5Etfw">@base</a><br><br>inspired from <a href="https://x.com/gakonst?ref_src=twsrc%5Etfw">@gakonst</a> tweets<br><br>here is how it looks like 👇 <a href="https://t.co/3o0OBUOm5f">pic.twitter.com/3o0OBUOm5f</a></p>&mdash; Dhruv (@0xdhruva) <a href="https://x.com/0xdhruva/status/1867176496542159103?ref_src=twsrc%5Etfw">December 12, 2024</a></blockquote> <script async src="https://platform.x.com/widgets.js" charset="utf-8"></script>
</div>

it blew up more than i expected. people liked the idea, shared use cases, gave feedback. first real validation that i was building something real.

## what i completely missed: the 402

here's the thing i didn't see until much later.

when a consumer hit an endpoint without a valid payment channel, my server returned an error — a 400 or 401. i had the right instincts about _what_ to communicate: "you need to pay to access this." but i never noticed the HTTP status code that had been sitting there for decades, designed exactly for this.

**402 Payment Required.**

dormant since the early web. waiting. and i was right next to it the whole time.

the irony: i _did_ use 402 for one case — insufficient balance. but i never connected the dots that it should anchor the entire spec. when coinbase launched x402 in may 2025, the thing that immediately clicked wasn't the spec itself. it was the name. they saw what i hadn't.

that's a useful lesson about how tunnel vision works when you're deep in building. i was so focused on the payment mechanics that i missed the protocol-level framing that would have made the whole thing click for people immediately.

## how the spec grew

between bangkok and the x402 launch, i kept extending pipegate.

i added one-time payment transactions — came out of a conversation with georgios about blockchain node snapshots. providers hosting snapshots for external node runners to sync faster. the fit was obvious: pay once, get access to the resource. works for datasets, reports, files, day passes, credit bundles.

i got into superfluid guild and worked on adding streaming payments. this opened up ongoing API access without monthly billing — live data consumption (market prices, IoT), extended compute or storage services that run over time. subscription-like without the subscription overhead.

by this point the spec felt complete: payment channels for concurrent usage, one-time tx for access tokens, streams for ongoing access. three modes, one middleware interface.

## when x402 launched

may 6, 2025. coinbase announced x402.

i read the spec the day it dropped. the 402 status code as the anchor, immediately obvious once you see it. the request/response format was clean and well-thought-out. genuinely good work on the standardization side.

what i didn't like: the relayer model, and the fact that it only supported token transfers. that's useful for a narrow set of cases, but it cuts out most of the use cases pipegate was built around streaming, one-time access, anything more nuanced than "transfer X tokens."

so i added x402 support to pipegate — but only the parts that made sense. 402 status code, same request/response format. but no relayer, and full payment method modularity intact. felt like the right synthesis.

## on timing and what actually matters

here's what building this early taught me about market timing.

being right about the problem doesn't mean being right about the moment. i spent three months on a spec that i believed in, in a space where the infrastructure — user onboarding, wallet abstraction, mainstream agentic adoption wasn't ready yet. the idea was right. the timing was early.

coinbase shipped x402 with simpler payment mechanics but infinitely more distribution. tempo launched MPP with stripe codevelopment and credit card + lightning support. georgios, whose tweet started this whole thing, was part of that. full circle.

what i learned: in protocol adoption, _simplicity of the initial spec beats completeness_. x402's v1 is deliberately narrow. that's a feature. pipegate tried to solve every payment mode at once, which made the idea harder to explain and harder to adopt even when the engineering was solid.

i also learned that early market signals are real and worth trusting. in late 2024, the pieces were all there — agentic infra growing fast, developers unhappy with API billing, onchain payments maturing. i was reading them correctly. i just didn't know yet that "reading the signal right" and "being the one who builds the standard" are two different things.

## where i think this goes

agentic payments are not a question mark anymore. the demand is obvious, more autonomous systems means more machine-to-machine transactions that need a payment layer. what's still unsolved is the onboarding layer: getting normal users (and normal developers) to a place where this is as easy as stripe.

the specs exist now. x402, MPP, pipegate's approach. the question is which abstraction layer wins for which use case, and whether the right UX gets built on top.

i'm still bullish. and i still think payment method modularity is going to matter a lot as the use cases get weirder and more varied.

**net result:** a real spec, a working library with 6.5k downloads, a lesson about tunnel vision (hi, 402), and a front-row seat to watching a space i believed in early start to matter at scale.

**if i redo this:** i anchor around the 402 from day one. i make the simplest possible version of the spec and ship that first. i think harder about distribution before i think about feature completeness.

here are some links if u'd like to read more about it -
_pipegate docs: docs.pipegate.0xdhruv.me — landing page: pipegate.0xdhruv.me_
