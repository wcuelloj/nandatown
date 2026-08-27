# Consensus scenario: a network partition freezes the leader instead of degrading the vote

## Scenario and setting changed

Scenario: consensus (1 leader, 19 followers, quorum 0.667)
Setting changed: failures.network_partition (task.config / failures block)

## Hypothesis, written before running

I come from product management in oil and gas, where field systems have to keep working even when part of the network goes fully dark, not just noisy or slow. That is what drew me to network_partition. The consensus scenario needs about 13 of 19 followers (2/3) to agree before a round commits. I split the 20 agents into two groups with a wall between them: the leader plus 9 followers on one side, and the other 10 followers on the other side. The leader can only reach 9 of the 19 votes it needs, which is below the quorum bar no matter how those 9 vote.

My prediction was simple: this should look like flipping a switch. The success rate should drop straight to 0, since there is no way to reach quorum with only 9 reachable followers. I expected a hard cliff, unlike the gradual decline you would see from something like a random 5 percent message drop.

## What happened

The prediction was wrong, but in an informative way. I ran the baseline first with no partition. It took 3 rounds out of the 5 allowed to reach quorum (round 1 got 63.2 percent acceptance, round 2 got 57.9 percent, round 3 finally hit 68.4 percent and committed).

With the partition in place, the run never got past round 1. The validator reported 0 committed rounds, but it still printed PASS, because the validator only checks rounds that actually committed, and with zero commits there is nothing to check. A broken run and a healthy run look identical to that validator.

The metrics report showed a success_rate of 64.29 percent, which looks like a near miss rather than a total failure. I checked the math by hand against the raw trace: 18 receive events out of 28 send events, 18/28 = 0.6429. That matches exactly, but it turns out this number has nothing to do with votes or agreement. It is measuring message delivery, not consensus.

I ran a second experiment as a control, giving the leader 14 of 19 followers instead of 9 (well above the quorum bar, at least by headcount). I expected this one to succeed. It did not. It also stopped after round 1, with a success_rate of 84.85 percent, another number that looks encouraging but still describes zero actual agreement.

## What surprised me and how I investigated it

The surprising part was that even the "easy" version of the experiment, with 14 of 19 followers reachable, still failed to commit anything and still stopped after a single round. That did not match either version of my prediction (full failure vs partial success), so I went into the installed package source to find out why.

In nest_core's consensus.py, the leader's vote handler waits for a message from every single follower before it will even score a round:

    if len(self._votes[rnd]) < self._num_followers:
        return

There is no timeout and no fallback. Under random message drop this is forgivable, since dropped messages are usually delivered eventually on retry. Under a network partition it is fatal, because the followers on the other side of the wall are not late, they are permanently unreachable. The leader waits forever for votes that will never arrive, so it never finishes scoring round 1, and the retry logic that would try rounds 2 through 5 never fires, no matter how many followers were actually reachable.

I then checked whether this same pattern shows up in the other scenarios by reading their source directly. Voting has the identical shape: the coordinator waits for every voter before announcing a result (voting.py). Auction has it too: the auctioneer waits for every bidder before closing a round (auction.py). Reputation and supply_chain do not have this problem, because they are built as one-on-one trades or a linear chain rather than one coordinator waiting on everyone at once. Marketplace does not have it either, for the same reason.

So the bug is not a one-off mistake in one file. It is a repeated design choice, "wait for all N participants, no timeout," that shows up in every scenario built around a single coordinator collecting responses from a whole group, and is absent from every scenario built as pairs or a chain.

## Robustness check across seeds

To rule out this being an artifact of seed 42 specifically, I reran the partitioned experiment (leader with 9 of 19 followers reachable) at four additional seeds: 1, 7, 100, and 999. Every seed produced the same result: 0 committed rounds. The freeze is a property of the partition itself, not a coincidence tied to one particular seed.


## Why this matters here specifically

Nanda Town's whole premise is that it tells you whether your protocol actually works. A validator that passes vacuously on zero events, and a metric named "success_rate" that is actually measuring message delivery, both make a frozen protocol look calm on the surface. Someone running this scenario under a partition and only glancing at the dashboard would see a percentage in the 60s to 80s range and a green validator, and could easily walk away thinking the system degraded gracefully, when in fact it never agreed on anything at all.

It is also worth noting that the maintainers already recognized the naming problem. Inside nest_core/metrics.py, the function used to be named _success_rate and was renamed to _delivery_rate, with a comment stating plainly that a 100 percent delivery rate does not mean the protocol succeeded. The old name is kept only as a backward compatible alias. But every built-in scenario YAML, including consensus.yaml, still requests the metric under the old, deprecated name, so the confusing label is what everyone actually sees by default.

## What I would build next

A timeout alone is not a full fix, and I want to be specific about why.

Adding a deadline would solve the hanging part: if a configurable amount of time passes without hearing from every participant, the leader stops waiting and scores the round using whoever answered.

What a timeout does not settle on its own is what the quorum should be measured against once you allow partial responses. Right now the code computes accepts divided by however many votes it collected, but it never gets there because it always waits for the full original roster of 19. The moment you add a timeout, you are forced to make an explicit choice that the current code avoids by simply never finishing:

- Quorum against the responders: 2/3 of however many actually answered (for example 2/3 of 9, about 6 votes). This keeps the protocol alive during a partition, but it means a proposal can commit with the agreement of a small, possibly one-sided slice of the group, which is a real trust question, not just a liveness one.
- Quorum against the original roster: 2/3 of 19, about 13 votes, regardless of how many could be reached. This is stricter and closer to what people probably assume "2/3 agreement" means, but it guarantees that any partition big enough to isolate more than a third of the followers makes agreement mathematically impossible, by design, rather than by bug.

Auction does not have this ambiguity, because its close condition is unanimity, everyone or nothing, so a deadline plus "clear with whoever bid" is a complete answer there. Consensus and voting use a fractional threshold, so a deadline is only half the fix. The other half is a policy decision the scenario currently makes by accident (it defaults to "never decide" rather than picking either option above), and I think that decision should be visible in the YAML as its own setting rather than left implicit in the code.

Alongside that, I would still add a validator rule that fails loudly, instead of passing vacuously, whenever zero rounds complete in a run that was configured to allow several. This is not a fix I am inventing from nothing: the repo's own hackathon material already shows the intended pattern (view_timeout_ticks and partition_heal_at_tick in scenarios/bft_consensus_partition.yaml), which is evidence the direction is sound rather than untested. A validator that cannot tell a stalled protocol from a working one is a separate problem worth fixing regardless of which quorum policy gets chosen.

## Related work

The auction scenario has a version of this same root cause: PR #88 ("one byzantine bidder halts all clearing") shows the auctioneer's unanimity-based close condition freezing after a single garbled bid, with no deadline, the same missing-timeout shape found here in consensus and voting. The difference is the trigger (a malformed message there, a network partition here) and the follow-on question consensus and voting raise that auction does not: once a timeout is added, should quorum be measured against the agents who actually responded or against the full original roster. Auction's rule is unanimity, everyone or nothing, so that question does not arise there.

## Tools and help used

Claude helped me survey open PRs to find interesting scenario/setting combinations, which led to network_partition on consensus and aligned with my oil and gas background where time is money and business must continue. I ran every command myself: baseline, partitioned, report generation, etc. Claude independently re-derived the reported numbers from the raw trace events, and I reviewed the same traces and confirmed the same numbers. When the control run did not match my prediction, Claude helped me read the nest_core source to find the actual cause, and helped check the other five scenarios for the same pattern. Claude also assisted with editing this writeup.

## How to reproduce

Baseline (no partition):

    nest run consensus
    python -c "
    from pathlib import Path
    from nest_core.validators import validate_trace
    for r in validate_trace(Path('traces/consensus.jsonl'), 'consensus'):
        print(('PASS' if r.passed else 'FAIL'), r.name, r.detail)
    "

Main experiment (leader partitioned with 9 of 19 followers, below quorum):

    nest scenarios cp consensus .
    # edit consensus.yaml: add a failures.network_partition block
    # group A: leader-0 plus 9 followers, group B: the remaining 10 followers
    nest run consensus.yaml -o traces/consensus-partitioned.jsonl
    python -c "
    from pathlib import Path
    from nest_core.validators import validate_trace
    for r in validate_trace(Path('traces/consensus-partitioned.jsonl'), 'consensus'):
        print(('PASS' if r.passed else 'FAIL'), r.name, r.detail)
    "

Control (leader partitioned with 14 of 19 followers, above quorum by headcount):

    nest run examples/consensus-partition-freeze/consensus-control-above-quorum.yaml
    python -c "
    from pathlib import Path
    from nest_core.validators import validate_trace
    for r in validate_trace(Path('traces/consensus-control-above-quorum.jsonl'), 'consensus'):
        print(('PASS' if r.passed else 'FAIL'), r.name, r.detail)
    "

Robustness check (same partitioned experiment, five seeds):

    nest run examples/consensus-partition-freeze/consensus.yaml --seed 1 -o traces/seed1.jsonl
    nest run examples/consensus-partition-freeze/consensus.yaml --seed 7 -o traces/seed7.jsonl
    nest run examples/consensus-partition-freeze/consensus.yaml --seed 42 -o traces/seed42.jsonl
    nest run examples/consensus-partition-freeze/consensus.yaml --seed 100 -o traces/seed100.jsonl
    nest run examples/consensus-partition-freeze/consensus.yaml --seed 999 -o traces/seed999.jsonl

    python -c "
    from pathlib import Path
    from nest_core.validators import validate_trace
    for seed in [1, 7, 42, 100, 999]:
        results = validate_trace(Path(f'traces/seed{seed}.jsonl'), 'consensus')
        for r in results:
            if r.name == 'consensus_agreement':
                print(f'seed {seed}: {r.detail}')
    "
