+++
title = "my experience through the wintermute alpha challenge 2025"
description = "sharing my journey and insights from participating in the Wintermute Alpha Challenge 2025."
date = 2025-09-09
updated = 2025-09-09
# draft = true  # Uncomment to keep as draft

[taxonomies]
categories = ["research", "blockchain", "smart contracts", "challenges"] 

[extra]

lang = "en"
toc = true  # table of contents
comment = false  # enable comments
copy = true  # copy code button
outdate_alert = true  # show outdated alert
outdate_alert_days = 120  # days before showing alert
math = false  # enable math rendering
mermaid = false  # enable mermaid diagrams
featured = true  # feature on homepage
reaction = false  # enable reactions
+++

![Wintermute alpha challenge 25 leaderboard](/wintermute-challenge-25/leaderboard.png)

## intro

this isn’t a polished technical writeup of every exploit, wintermute already publishes great case studies for that. instead, this is my own devlog-style account of how i approached the alpha challenge 2025: the tools i used, the way i thought through problems, where i got stuck, and what i learned along the way.

i first heard about the challenge from someone who competed last year and later joined wintermute, which got me curious. i expected more research-analyst style questions (finding data, doing protocol-level case studies, etc.), but it turned out to be much more hands-on: debugging exploits, replaying attacks, decompiling bytecode, liquidations, and even a pvp liquidity game.

this mix of research + execution hooked me instantly. i’m not a security researcher by background, but i like solving puzzles, and my time as a research engineer at stackr labs taught me to learn by doing. this challenge became a sprint of learning defi internals under pressure.

if you want the solutions, check this [case studies writeup](https://github.com/Frodan/wintermute-alpha-2025-writeups) by @frodan. if you want to see how someone like me worked through them, right or wrong, then read on.

## tools i used

before diving into the case studies, here’s what i leaned on the most throughout the challenge:

- **foundry (cast & forge)** → my main swiss-army knife for onchain lookups, scripting transactions, and replaying attacks on forked mainnet.
- **etherscan decompiler** → surprisingly useful for challenges without verified source. being able to peek into opcodes and constants gave me just enough context to patch things.
- **dune dashboards & explorers** → for quick market / protocol context. sometimes just scanning the right chart gave me clues.
- **gpt 5** → not for exact solutions (it was usually clueless about solving these puzzles), but for brainstorming, research, confirming syntax, and occasionally pointing me to the right docs.
- **scratch notes + manual trial & error** → honestly, a lot of this challenge came down to debugging small mistakes: wrong address, proxy vs implementation confusion, or under-estimating liquidity.

having these at hand gave me speed, even if i didn’t know the protocol or exploit beforehand.

## obstacle race

this was the first real puzzle i tackled. the task was to “modify the bytecode and test in forked env.” at first i didn’t get it, how do you even change deployed bytecode? but then it clicked: the goal was to frontrun the attacker and deploy our own patched contract to rescue funds.

i had never played with raw evm bytecode before. i used etherscan’s decompiler first, then dedaub decompiler to reverse the contract and noticed every function checked against a double-hashed `tx.origin`. the hash constants looked incomplete at first, until i realized they were split across two push opcodes: `32 bytes → 1 + 31 bytes` which was the tricky part in this challenge

after a few failed attempts, i patched the constants with my own address, redeployed, and replayed the attack successfully.

**takeaway:** this was my crash course in how far you can go with just bytecode + opcodes. i learned that even if the source isn’t verified, you still have options.

the same concept applied to the “rabbithole” challenge as well

## before the storm

this one revolved around liquidating a position in llamalend after the uwu lend hack. at first, i didn’t know llamalend’s flow, and the address provided in the challenge didn’t map clearly in my head.

so my first step was just researching about the protocol: what llamalend is (a curve sub-product), how positions are tracked, and what exactly needed to be liquidated. once i had that clarity, the path became clear: i needed a flashloan to get curvUSD, convert, liquidate the position, and walk away with CURV.

the real pain here wasn’t the logic, it was juggling liquidity and failed attempts. i first tried flashloaning CURV directly from curv’s flashlender (wrong, not deployed at that block). then i switched to balancer for flashloan and swaps `USDC → curvUSD → CURV`. the amounts were large enough that full liquidation wasn’t always possible, so i had to settle for partials. still, i managed to bag ~95k CURV.

**takeaway:** working across protocols isn’t just “write contract and done.” the actual liquidity available at block height can break the flow.

## it’s oiler

the name gave me a laugh once i realized it was “euler” in a scottish accent. i thought this would be simpler than others, but it ended up eating a ton of time.

the challenge gave me a huge eWETH collateral position. my job: extract as much value as possible. at first, i thought i’d just borrow against it. but in reality, it turned into a search problem, finding which markets still had liquidity worth borrowing as most of them were drained after the hack.

i wrote a script to scan all euler markets, filter out useless tokens, and find pools with real balances. then came the swapping part. uniswap v3 alone wasn’t enough, so i had to piece together multi-hop routes on v3 across different fee tiers, plus sushi/pancake.

i ended up splitting my collateral across multiple contracts to get around isolated borrowing restrictions. in total: 27 txs, ~3.8m recovered. not the most optimized work i could’ve gone for, but it got the job done.

**takeaway:** i learned:

- scanning/token filtering via scripts is critical, saves hours of manual trial.
- sometimes the “brute force” approach (multiple contracts, many txs) is fine if you’re under time pressure.

## pvp competition

this was unlike the others: not a puzzle, but a live strategy game. we had custom tokens with univ3 pools at different fee tiers, and wash trading was driving volume. the way to win was provide liquidity in the right ranges and farm fees or do FCFS challenge.

i debated writing a script but with only 3–4 days left, i went manual. i inspected pools, spotted low-liquidity but decent-volume ones (like `acUSDT/acAlpha 0.05%`), and dropped liquidity there. i also tested the bigger `0.3%` pool.

my routine became: monitor ranges, swap between acUSDT ↔ acAlpha, reposition liquidity, and try not to bleed too much in fees myself. it was messy, sometimes i got pushed out of range, sometimes i lost on swaps, but i did end up earning a fair chunk of fees before losing it all at the end again 😅

**takeaway:** unlike the other challenges, this was about real-time decision making. i learned how hard active liquidity management is in practice, and how quickly fees you pay can eat into gains.

## other highlights

- **dark amms** → this was the first written case study i touched, but i didn’t finish during the challenge window. i learned that “dark” here didn’t mean privacy but hidden pricing logic: market maker operated contracts updated every block, no public frontends. the concept fascinated me because it’s so well-suited to solana’s low-latency blockspace, and i kept thinking about how something similar might work on L2s with flashblocks. even unfinished, it sparked new ideas for me.
- **what if** → this was the challenge i sunk the most time into without a neat solution. the task was open-ended: find skewed pools, rescue as much stETH as possible. i mapped the main dexes (curve, balancer, uniswap), spotted imbalances, and drafted scripts for cross-dex swaps. but execution was messy - low rates, failed swaps, partial arb opportunities. with just a few hours left, i chose to move on.
- **too old for optimising** → this one clicked late but fast. i dug into yearn’s yUSDC vaults, realized rebalancing could push funds(~700k USDC) into aave v1, and then leveraged my existing usdt collateral to top up the usdc pool. it wasn’t the cleanest trick, but it worked and got me across the line.

## final thoughts

across all the challenges, i made plenty of mistakes, using wrong addresses, mixing up proxies, chasing liquidity in the wrong pools. but then I slowed down, did debugging, and figured things out.

the biggest things i took away from this sprint:

- **tools > knowledge** → knowing how to use cast, decompilers, explorers mattered more than memorizing protocol docs. with ai tools like chatgpt and claude code, the process becomes simpler and can be sped up.
- **process matters** → mapping a protocol before touching contracts saved me from wild goose chases.
- **curiosity wins** → not being a “security researcher” didn’t matter. being willing to try, fail, and try again mattered more.

i loved participating in alpha challenge 2025. it pushed me into bytecode, flashloans, liquidations, and liquidity management in ways i hadn’t done before. more than anything, it showed me how much i enjoy this type of puzzle-solving. i ended up ranking 17th among 1000+ participants in the leaderboard with a score of 500, not bad ig. i’ll definitely keep joining these competitions and keep learning in public.

thanks to wintermute for organising, and to everyone who shared their own approaches.
