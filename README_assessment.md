# Voting under Byzantine agents

**Author:** Josh Beasley

**Scenario:** `voting`

**Setting changed:** `failures.byzantine_agents`, `0.0` to `0.2`. 

## Hypothesis

Delivery-oriented metrics would stay flat at 100 percent while the vote outcome silently degraded, with severity depending on which roles the injector happened to hit. I expected a corrupted tally with traffic roughly unchanged.

## Evidence

| | baseline | byzantine 0.2 |
|---|---|---|
| rounds completed | 5 | 1 |
| events | 410 | 112 |
| messages | 370 | 72 |
| `unique_pairs` | 37 | 36 |
| corrupted receives | 0 | 4 |
| `delivery_rate` | 1.0000 | 1.0000 |
| `dropped_count` | 0 | 0 |
| `success_rate` | 1.0000 | 1.0000 |
| `agent_count` | 20 | 20 |

In the baseline, coordinator-0 received 90 votes and sent 5, one report per round back to proposer-0, and the run completed its five rounds.

Under corruption the run stopped after one round. All 18 proposals arrived intact. Four votes arrived corrupted at coordinator-0, from voter-6, voter-8, voter-10 and voter-12, which is exactly 0.2 of the twenty agents. Coordinator-0 received all 18 votes, emitted nothing, and all twenty agents then logged a normal `stop`. The single missing edge in `unique_pairs` is the coordinator-to-proposer report that drives the next round.

Every health metric is identical across the two runs. Corruption is not loss, so nothing registered as dropped, and the run reports as fully successful having completed one round out of five.

## Investigation

I expected a corrupted tally. I got a halt. Filtering the trace for non-printable payloads turned up two things.

**Corruption is applied in transit.** Voter-6 logged `vote:1:no:voter-6` on send. Coordinator-0 logged scrambled bytes on receive, same correlation ID, same 17-byte size. Not one send event anywhere in the trace is corrupted. An audit of sender-side logs alone would show a clean, fully participating election, and the byte lengths are preserved exactly, so any size-based integrity check passes.

**The failure is orderly, not loud.** The coordinator consumed all eighteen votes, produced no output, and the run shut down normally. The trace schema has no error event kind, so a dead-ended tally is indistinguishable from a completed one. `success_rate` reports 1.0000 for a run that finished twenty percent of its work, because it measures delivery rather than whether the vote resolved.

## Use of AI and other help

Claude for drafting some of the textual answers and this README. Claude Code for the local runs and for pulling structure out of the JSONL. I ran the scenarios, generated the reports, and read the traces myself. 