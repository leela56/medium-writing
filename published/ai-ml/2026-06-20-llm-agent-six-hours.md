# My LLM Agent Ran for Six Hours. It Did Nothing Useful. That Was My Fault.

**Published:** 2026-06-20
**Medium:** https://medium.com/@leelasaikiran4/my-llm-agent-ran-for-six-hours-it-did-nothing-useful-that-was-my-fault-d43cdfbf99f3
**Read Time:** ~7 min
**Tags:** ai-agents, llm, agentic-ai, prompt-engineering

---

*Autonomous agents don't fail because the models are bad. They fail because nobody defines what "done" means.*

I ran an agent loop for six hours in early March. The task was simple on paper: read through a backlog of GitHub issues I'd accumulated on a side project, group them by root cause, and produce a prioritized write-up I could actually act on. I gave it tool access to the repo, a write-back tool for saving drafts to disk, and a prompt that felt thorough at the time. I went to lunch. When I came back, the agent had created 47 sub-tasks, written 12 draft summaries, deleted three of them, and was mid-way through re-reading issues it had already grouped. The token bill hit my personal API limit by 4pm.

The model wasn't confused. It was doing exactly what the prompt implied it should do.

That's the problem.

## The architecture that made sense until it didn't

The setup was standard. A planner agent broke the issue backlog into chunks. A reader agent processed each chunk and extracted themes. A writer agent compiled the themes into a summary. I'd seen this pattern work on shorter tasks, so I scaled it up.

The first thing that broke was **termination**. The planner didn't know when it had planned enough. I had written "identify all meaningful clusters" as the success condition, and the model interpreted "all" and "meaningful" as requiring continued verification passes. Not because it was broken — because "all" and "meaningful" are genuinely ambiguous instructions and the model did what any reasonable interpreter of those words would do: kept checking.

The second thing that broke was **memory**. Each agent call was stateless. The reader agent had no idea what the planner had already confirmed. So it occasionally re-derived conclusions the planner had already logged, because from its perspective it was starting fresh. The outputs were consistent with each other. They were just redundant, and the orchestrator had no rule that caught redundancy as a stop condition.

I had built an agentic system without defining what "done" looked like. Not approximately done. Not "mostly complete." **The specific condition under which the loop should halt and return.**

It failed. Not because the model hallucinated. Not because the tools were broken. It failed because I treated "done" as obvious.

## What I actually fixed, and what it cost

The fix was boring. I added an explicit task registry that each agent wrote to on completion. The planner could check the registry before spawning a new subtask. The writer agent checked whether a summary for a given cluster ID already existed before starting. The orchestrator compared the registry state against a simple completion predicate before allowing another loop.

This felt like plumbing. **It's actually the whole job.**

The completion predicate was the part I got wrong twice before getting right. My first attempt was "all issues have been assigned a cluster." That broke because some issues genuinely don't cluster, and the writer agent would get stuck trying to assign them. The second attempt added an unclassified bucket, which was closer. The third attempt also specified a maximum loop count per cluster, which is what finally made the agent stoppable in a predictable amount of time.

The maximum loop count felt like cheating when I added it. **It's actually just correct. Agents that can run indefinitely will run indefinitely.** The question isn't whether to bound them, it's where to put the bound.

The total time to rework the termination logic was maybe four hours. The six-hour runaway loop cost more in tokens than the fix cost in engineering time.

## The three agent patterns I've seen in the wild

They cluster into three patterns, though these shade into each other in practice.

**The pipeline agent.** Fixed steps, linear handoffs. Fine for narrow tasks. Breaks the moment anything needs to loop back, because the pipeline has no concept of that and the model improvises.

**The planner-executor pair.** A planner generates a task list; an executor works through it. Most mid-complexity systems land here. It breaks on two things: the planner generates tasks that can't actually be executed independently, and the executor has no authority to say the plan is wrong. If the plan is bad, the executor completes every task on it and reports success.

**The fully recursive agent**, which can spawn sub-agents, wait on results, and branch based on what comes back. This is what I was running. It's also where the six-hour loops live. **Recursion without a completion predicate and a depth limit is a while-true loop.**

The thing none of these patterns solve natively is the question of when the agent should stop and ask. Every agentic system I've built or read about treats "ask for human input" as an edge case. It's usually the thing you reach for after the loop has already done something unfortunate.

## What I still don't have answers for

**Confidence thresholds for stopping.** I've added loop counts and completion predicates, but I haven't figured out how to make the agent stop when it's *uncertain* rather than when it's finished. The model will tell you it's uncertain if you ask. Building that into a structured stop condition — one that's neither too sensitive nor too permissive — I don't have a clean approach. I've been experimenting with asking the agent to score its own confidence on each output and halting below a threshold, but the self-scores are noisy and calibrating the threshold is mostly vibes right now.

**Multi-agent trust.** When the writer agent gets output from the reader agent, it trusts it completely. That's fine when the reader is working correctly. It's not fine when the reader has misclassified something and the writer builds a whole summary on top of a bad cluster. The pipeline doesn't have a step that says "check the upstream output before using it." Adding one means the whole thing slows down substantially. I'm not sure the tradeoff is always worth it.

## Agents don't need better models. They need better contracts.

Most of the tooling investment right now is going into making models smarter: longer context windows, better multi-step reasoning. Those improvements matter. They don't fix the category of failure I keep running into.

The category is this: **the system knows how to do the work, but nobody told it when the work is over.** That's not a model problem. That's a spec problem. A smarter model that runs indefinitely on an ambiguous completion condition is just a more expensive version of the one I already had.

What actually changes agent behavior is a completion predicate that the system can evaluate without a human watching. What state means done. What condition means stuck, not just slow. Those aren't model questions. That's spec work, and it's tedious, and it's the difference between a system that ships and one that generates 47 sub-tasks and a non-trivial token bill while you're at lunch.

---

*If you've shipped an agentic system in prod and figured out a principled way to handle confidence-based stopping, or multi-agent output verification that doesn't kill throughput, I'd take a better answer than the one I have.*
