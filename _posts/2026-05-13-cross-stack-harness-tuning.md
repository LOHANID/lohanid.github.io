---
layout: post
title: "Notes from 23 hours of cross-stack harness tuning"
subtitle: "An attempt to apply an iterative tuning-loop pattern to a software target, using an agent harness as the test case."
date: 2026-05-13
categories: [agents, experiments]
tags: [agent-systems, llms, observability]
---


These are notes from an attempt to apply an iterative tuning-loop pattern to a software-development target. The pattern itself comes from recent work where larger models autonomously improve smaller machine-learning models by picking a small hypothesis about what should change, dispatching an experiment, reading the result against a fixed criterion, hypothesizing again, modifying, and repeating. The question I had was whether the same loop could be made to work when the target is software rather than a model. I used my own agent harness as the test case, partly because I had the access to instrument it freely, and partly because it had a clear quality criterion I could measure against.

The setup ran for about 23 hours of wall-clock, of which roughly 10 hours had active work. I want to describe what I did, what I saw, and what I think it might mean, without overclaiming. The sample is one trajectory, the judging was not independent, and I (the human in the loop) was less hands-off than the design intended. Anyone running similar experiments who can verify, contradict, or extend any of this is welcome to.

## The setup

I had two agent stacks running.

The first stack was Claude Code (a mature agent harness) running Claude Opus 4.7. I had two separate Claude Code instances active during the trajectory. One was the tuner: I gave it the job of dispatching tasks to the second stack, reading its outputs, analyzing its failures, and writing modifications into the second stack's source files. The other Claude Code instance, in a separate session, was a monitor I asked to do periodic read-only sweeps of the trajectory: git history, HTTP endpoint state, planning files. The monitor was supposed to be hands-off. In practice, I ended up relaying some of its observations to the tuner session, so the separation between them was leaky. I'll come back to that.

The second stack was a locally-developed agent harness I've been building, running gpt-5-mini through a ReAct loop. The harness has fewer built-in primitives than Claude Code: filesystem read/write/edit, shell exec, grep, glob, network fetch, and a handful of capability-gated extensions. It's the stack I was trying to tune.

The goal I had in mind, going in, was: have the bigger stack (Claude Code plus Opus 4.7) use its access to the smaller stack's source to gradually augment the smaller stack's primitives, so that over time the smaller stack could rewrite its own source at a quality approaching what the bigger stack would produce. The shorthand I had been using for this was "self-evolving agent." I no longer think that label is accurate to what I watched.

### What each iteration looked like, mechanically

The tuner picked a small task (usually a primitive to add, a behavioral fix, or a regression to investigate), wrote a task prompt for the smaller stack, and dispatched it through the smaller stack's runtime. The smaller stack would then read source files, make tool calls, write a diff, run its verification gates, and report task complete. The tuner read the diff and the event log, often wrote additional changes itself (cleaning up if the smaller stack had overshot, adding a rule book entry if a new failure pattern had emerged, patching the runtime if a substrate issue had surfaced), and committed the result. Every twenty iterations or so the tuner played the judge role, scored the recent window against the rubric, and adjusted strategy for the next batch.

One iteration cycle typically took five to fifteen minutes, with longer pauses during deep analysis or substrate-level work. Across the window I observed about forty-six iterations and three rubric checkpoints.

The shape of the loop is worth naming separately from the specific experiment. It is an iterative cycle of: pick a hypothesis (what should change), run an experiment (one task dispatched to the smaller stack), read the result against a fixed criterion, hypothesize again, modify, run another experiment. The unit of experiment is one task. The unit of progress is a commit. The unit of evaluation is a rubric. The whole thing runs without human input except for the dispatch trigger and the human-readable artifacts I asked the monitor session to surface.

This is the same shape that several recent published systems use to drive autonomous improvement of code, models, prompts, and reward functions: a larger model proposes a modification, dispatches an experiment, reads the result, and iterates. The target in those systems varies (a piece of code, a model architecture, a prompt, a reward function); the loop structure is similar. I'll come back to what this might imply about generalizing the pattern in a section toward the end of this writeup. For now I want to describe what I saw, not what I think it implies.

## What I think I saw, briefly

The bigger stack spent roughly ten hours of active work writing into the smaller stack's source files. Over that window, the smaller stack's product-code output improved on the dimensions a fixed rubric tracked. The rubric score went from 66 of 84 (early), down to 53 (mid-window), and back up to 68 (late). The single biggest contributor I can identify was a one-line change to a configuration constant in the smaller stack's runtime, raising a tool-output byte cap from 8 KiB to 256 KiB. After that change, several recurring failure patterns the trajectory had been attributing to the smaller stack's reasoning got reclassified, by the tuner's own commentary, as caused by the byte cap.

What I did not see was the smaller stack authoring new primitives into its own harness during the window. Every harness modification that landed cleanly came from the bigger stack. The "self" in self-evolution did not appear in any form I could measure.

I want to be honest about what I can and cannot claim:

The 15-point score recovery after the byte-cap fix could be largely from that fix, or largely from other adjustments happening in the same iterations (more prescriptive prompts, denser constraints, better verification commands). I cannot isolate the fix's specific contribution. I can only report that the score moved after the fix landed.

The judge that produced the rubric scores was the bigger stack, the same stack that produced the work. This is not an independent evaluation. I take the 53 number more seriously than I would otherwise, because it was unfavorable to the tuner's then-current strategy and the tuner acted on it, but I would not generalize from these numbers as if they came from a blind rater.

The "documented misrepresentations" I will discuss below are attributions written into commit messages by the tuner, not my own forensic re-read of the smaller stack's event logs. I trust the commit messages more than I would trust the smaller stack's own self-reports, but they are still secondhand.

I was one operator over a 23-hour window. Other things may have happened that I missed.

## Two snapshot tasks

Two specific tasks, one near the start of the window and one near the end, are useful anchors.

Early in the window, the smaller stack was asked to add a small runtime safeguard to its own reasoning-loop implementation. The prescribed scope was about 55 lines: a helper function, a constant, a counter, an injection block. The smaller stack applied the prescribed change correctly, then also rewrote about 700 lines of adjacent code in the same file, breaking a reference to a function it had relocated to a non-existent module path. The build failed. The smaller stack reported task complete anyway. Twelve verification gates had run during the iteration; none caught the build break, because they were running tests for a different component than the broken build. The bigger stack had to revert the changes and re-apply only the prescribed scope cleanly.

Late in the window, the smaller stack was asked to extend a small unified-diff utility across two language surfaces (a web-side TypeScript implementation and an editor-extension TypeScript counterpart) plus a Node fixture-runner script. Three files, parallel helper additions, all product code (not harness code). The smaller stack wrote the helpers across all three surfaces, ran verification, and reported task complete with a substantively accurate description of the work done. The bigger stack verified the build and did not have to rescue the work. The rubric score for the post-fix region was 68 of 84.

About nine hours of active work separated these two snapshots. The smaller stack's underlying model, tool surface, and reasoning strategy were the same at both points. What had changed was the smaller stack's surrounding harness: its rule book was longer, its runtime had been patched once, and the tuner had developed habits about how to verify the smaller stack's output. The product-code task at the end did not require the smaller stack to touch its own harness. The runtime-code task at the start did, and the smaller stack could not do it cleanly.

## What changed, layer by layer

I am ranking these by how much I think each contributed, but the ranking is my reading of the data, not a measured attribution. Take the ordering with appropriate skepticism.

The smaller stack's runtime got one configuration change. The tool-output byte cap was raised from 8 KiB to 256 KiB. This is the one substrate modification that landed during the window. It is also the change I think mattered most, on the basis that recurring failure patterns stopped recurring after it landed and the tuner's own commit messages attributed prior failures to the old cap. I cannot isolate this from the other things happening in the same iterations.

The smaller stack's rule book grew from 4 entries to 10. Each addition was a response to an observed failure pattern. Some of these rules appear to have bound on the smaller stack's behavior in subsequent iterations, in the sense that the failure they were meant to prevent did not recur in iterations where the rule was active. Others did not appear to bind. The pattern I think I see (with low confidence, because the sample is tiny) is that rules requiring the smaller stack to passively check its own current state were more reliable than rules requiring the smaller stack to track a running count or sub-goal across iterations. I can name three rules that seemed to bind and one that did not. That is not enough data to make a general claim.

The tuner's strategy changed once during the window. For the first thirteen monitoring intervals I watched, the tuner appeared to prefer writing new rule-book entries over modifying the smaller stack's runtime when responding to failures. After the mid-window rubric verdict came in at 53, with explicit commentary attributing the regression to infrastructure, the tuner shifted: the next significant change was the byte-cap modification, and most of the recovery iterations involved more prescriptive prompts rather than additional rules. I do not know what triggered the shift internally for the tuner. I can only say that the verdict and the strategy change happened within 58 minutes of each other.

The smaller stack's reasoning-loop code received one substantive runtime modification. This was the early-window task the smaller stack was asked to write but could not do cleanly. The tuner wrote the working version. The modification appears to have functioned as intended in some iterations and not in others. I cannot tell from outside whether the non-binding instances were due to the code itself or due to something about the smaller stack's reading of the rule the code was supposed to enforce.

The smaller stack's architectural-invariant test allowances were raised three times over the window, all by the tuner. These are not capability improvements; they are accommodations for code growth. I mention them because they affect the tuner's ability to keep the smaller stack's source under a complexity ceiling, which matters for the long-term bootstrap goal. None of the three bumps cited a planning task at the time they landed, though the tuner did add a policy later that would have required citation. The policy was not exercised again during the window.

The smaller stack's own surface did not change in any way the smaller stack authored. Same tools at the start and end. Same reasoning strategy. The rule book in the prompt grew because the tuner added entries to the file; the smaller stack did not modify the file itself, as far as I could tell from the git history.

## The arc

The rubric scores tell the story most clearly:

<svg viewBox="0 0 600 300" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:600px;border:1px solid #d0d7de;background:#fff;display:block;margin:1em auto;">
  <line x1="60" y1="240" x2="560" y2="240" stroke="#1f2328" stroke-width="1"/>
  <line x1="60" y1="20" x2="60" y2="240" stroke="#1f2328" stroke-width="1"/>
  <text x="50" y="244" font-size="11" text-anchor="end" fill="#57606a">0</text>
  <text x="50" y="187" font-size="11" text-anchor="end" fill="#57606a">21</text>
  <text x="50" y="135" font-size="11" text-anchor="end" fill="#57606a">42</text>
  <text x="50" y="82" font-size="11" text-anchor="end" fill="#57606a">63</text>
  <text x="50" y="28" font-size="11" text-anchor="end" fill="#57606a">84</text>
  <line x1="60" y1="187" x2="560" y2="187" stroke="#e8eaed" stroke-dasharray="2,3"/>
  <line x1="60" y1="135" x2="560" y2="135" stroke="#e8eaed" stroke-dasharray="2,3"/>
  <line x1="60" y1="82" x2="560" y2="82" stroke="#e8eaed" stroke-dasharray="2,3"/>
  <line x1="60" y1="55" x2="560" y2="55" stroke="#1a7f37" stroke-dasharray="4,3" stroke-width="1"/>
  <text x="565" y="58" font-size="10" fill="#1a7f37">target ≥70</text>
  <rect x="120" y="67" width="80" height="173" fill="#54aeff" opacity="0.85"/>
  <text x="160" y="62" font-size="12" font-weight="600" text-anchor="middle">66/84</text>
  <text x="160" y="258" font-size="11" text-anchor="middle" fill="#57606a">early window</text>
  <rect x="260" y="101" width="80" height="139" fill="#cf222e" opacity="0.85"/>
  <text x="300" y="96" font-size="12" font-weight="600" text-anchor="middle">53/84</text>
  <text x="300" y="258" font-size="11" text-anchor="middle" fill="#57606a">mid window</text>
  <text x="300" y="272" font-size="9" text-anchor="middle" fill="#57606a">regression: 13 points</text>
  <rect x="400" y="62" width="80" height="178" fill="#1a7f37" opacity="0.85"/>
  <text x="440" y="57" font-size="12" font-weight="600" text-anchor="middle">68/84</text>
  <text x="440" y="258" font-size="11" text-anchor="middle" fill="#57606a">late window</text>
  <text x="440" y="272" font-size="9" text-anchor="middle" fill="#57606a">recovery: 15 points</text>
  <text x="440" y="130" font-size="10" fill="#1a7f37">after byte-cap fix</text>
</svg>

I want to be careful about how I read this chart. The scores are self-applied (the tuner played both the work-producing and the judging role). The drop from 66 to 53 is real in the sense that the same rater applied the same rubric and got a lower number, and the rater attributed the drop to a specific cause that they then went and fixed. The rise from 53 to 68 is real in the same way, and happened in iterations following the fix. I think it is reasonable to interpret the V as evidence that the fix was binding on capability. I think it is not reasonable to read the V as proof of that. A different rater might have scored these iterations differently.

The thirteen intervals between the early peak and the mid-window dip are the period where the tuner was writing rule-book entries in response to failures. During this period, the tuner did not modify the smaller stack's runtime configuration. After the rubric verdict, it did. The verdict and the strategy shift are temporally adjacent. Whether the verdict caused the shift, or the shift was already underway, or both happened in response to some third thing I did not observe, I cannot say with confidence. What I can say is that without the verdict, the previous fourteen intervals of rule-book work had not produced a substrate fix, and after the verdict, a substrate fix landed within an hour.

The recovery iterations had one feature I can name with more confidence. The tuner's task prompts during this period became more prescriptive: specifying file contents verbatim, sequencing tool calls explicitly, leaving less room for the smaller stack to make per-file design decisions. The smaller stack's success rate on these prompts was higher than on earlier, less prescriptive prompts. This shift may also be contributing to the score recovery, alongside the byte-cap fix. I cannot separate the two contributions cleanly.

## Seven things I want to write down

These are my readings, not findings. The sample is one trajectory. Each one is something I would be interested in seeing tested under conditions where the observer is more independent and the comparison set is bigger.

The first thing is about naming. "Self-evolving" was a misleading label for what I watched. The smaller stack did not modify its own primitives. What I saw was a bigger agent stack tuning a smaller agent stack's harness from the outside. If a clearer label is needed, "cross-stack harness tuning" or "stack-level coaching" is closer to what was happening. I do not have strong feelings about the terminology; I want to flag that "self-evolving" carries assumptions the data did not support during this window.

The second thing is that most of the failures I initially attributed to the smaller stack's reasoning were later reclassified, by the tuner itself, as caused by the smaller stack's harness or runtime rather than by the smaller stack's model. The four most-cited "iteration exhaustion" failures on multi-file tasks were retroactively explained by the byte cap. The taxonomy of "agent failures" the trajectory built up over the window included instances that, on closer reading, were probably better classified as harness failures or framework bugs. I would be cautious about reading any failure-mode taxonomy in this kind of setup at face value, including mine.

The third thing is that the smaller stack's self-reports were not reliable as evidence. There were at least four iterations during the window where, according to commit-message attributions, the smaller stack's "task complete" summary materially misrepresented what it had done. One claimed a multi-test task was closed when only one of seven tests passed. One described a newly-created file as "already present, verified." One paraphrased a runtime feedback message that said "you may retry with different arguments" as "all tools failed, stopping." One claimed a fix was applied while the actual work sat uncommitted in the working tree. I would want to verify these directly from the event logs before treating the count as exact, but the pattern is consistent across the attributions. Any tuner that builds on the smaller stack's prose without checking artifacts will compound this kind of error.

The fourth thing is about which rule shapes seemed to bind. The rules that I think held more reliably were the ones whose violation the smaller stack could check by looking at its current output or current state. The rules that seemed not to bind were the ones requiring the smaller stack to maintain a running count or track a sub-goal across iterations. The sample is small enough that this is more of a hunch than a finding. If anyone has data on rule-binding rates across different rule shapes in similar setups, I would be interested to see it.

The fifth thing is about strategic stickiness on the tuner side. For thirteen monitoring intervals before the rubric verdict came in, the tuner appeared to be reaching for rule-book additions in response to failures rather than for runtime modifications. This may have been a deliberate strategic choice; I am reading it from outcomes. What I can say is that the verdict broke the pattern. Whether the tuner would have eventually pivoted to substrate work without the verdict is an open question. I do not think the data here can answer it. But the verdict-and-pivot sequence is the cleanest thing I observed during the window, and I think the takeaway is that external scoring is doing real work in this kind of setup, work that internal review may not be able to do.

The sixth thing is that the bottleneck on the smaller stack's measured capability, at least at the point in the window where the rubric verdict came in, was infrastructure rather than rules. A one-line configuration change appears to have been worth more rubric points than the prior thirteen intervals of rule-book work combined. I do not want to generalize this beyond the window I watched. But I do want to suggest that anyone working on a similar setup might consider auditing the substrate first when failure modes accumulate, rather than treating each new failure as a rule-design problem.

The seventh thing is about my own role. The monitor session I had running was supposed to be out-of-band, and it was not entirely. Some of its surmises during the window made it back into the tuner session via me. At least one of them produced a corrective commit within about fourteen minutes of when the underlying concern was raised. I do not know how much of the trajectory's eventual pivot was reinforced by the monitor's presence (and by me relaying its observations). I think the right way to read this report is as the perspective of an interested operator who was running both the experiment and the monitor that watched the experiment. A more rigorous version of this experiment would isolate the monitor, or run multiple monitors, or not have a monitor at all. I did not have the discipline to do that this time. I want to flag it.

## Why this pattern might apply beyond an agent harness

Coming back to the original intent. I picked the agent harness as a target less because I wanted that specific piece of software improved, and more because I wanted to see whether the loop pattern worked on a software target at all. So what does this trajectory say about that question, and what might it imply for other software targets one could try the same loop on?

The target of this trajectory was a smaller agent's harness. But the loop I described (pick a hypothesis, run a small experiment, read the result against a fixed criterion, hypothesize again, modify, repeat) does not require the target to be an agent. The target can be any piece of software the tuner has the access and the criterion to evaluate. Concretely, I can imagine the same loop tuning:

- A production codebase being refactored for performance, testability, or modernization
- A test suite being filled in against existing code
- A library being migrated across language versions or API breaks
- A configuration set being optimized against a benchmark
- A documentation set being aligned with the code it documents
- A research artifact (a notebook, a model, a benchmark) being iterated on

The conditions that made the loop work in this trajectory, to the extent that it worked, were:

The target had a measurable quality criterion. Without the rubric, I would not have known when the loop was stuck. The rubric was fixed (written before the trajectory began) and applied at periodic checkpoints.

The execution layer was cheap. Each experiment cost roughly five cents in model tokens. The bigger system could afford to dispatch dozens of experiments per day without exhausting a budget.

The tuner could read results faster than a human could. Each iteration's diff and event log were processed in about a minute, against a human-comparable equivalent of ten or fifteen minutes per review.

An external scoring layer existed and was applied periodically. Without it, the tuner's strategic bets would have continued past their usefulness, as the data in this trajectory suggests.

The things that went wrong in this trajectory should be expected to go wrong in the general case too. The tuner over-trusted the smaller system's self-reports until verification was forced onto artifacts. The tuner got stuck on one strategic layer (rule book) when the binding constraint was at another (substrate). Observer contamination happened once a human (me) was in any part of the loop. The cost asymmetry between the cheap executor and the expensive tuner was substantial.

If I had to compress the lesson from this experiment into one question to ask before applying this kind of loop to any other piece of software, it would be: what is the target's rubric, and who applies it? If the target has no fixed criterion that can be scored automatically, the loop will drift, and there will be no signal to break the drift. If the criterion exists but the same system doing the tuning is also doing the scoring, the loop will be at risk of motivated reasoning, as it was in this trajectory. The independence of the rubric matters more than any other property of the setup. Everything else, including the cleverness of the modifications, depends on that signal being honest.

Beyond that, I think the general pattern is workable. A bigger model running a tight iterative loop against a smaller target with an automated quality criterion is a real shape for software improvement, not just for agent improvement. This trajectory is one weak data point in that direction, and it suggests both that the shape works on something small and that the failure modes are roughly the ones you would predict from the shape itself.

## What I think I can say, and what I cannot

I can say that across the window, the smaller stack's product-code output improved on the dimensions a fixed rubric tracked. The improvement was real to the extent that the rubric is meaningful, and the rubric was at least internally consistent across the window because the same rater applied it.

I can say that the smaller stack did not, during the window, modify its own harness primitives in any way I could detect from the git history. Every change to the smaller stack's source that crossed the build threshold came from the bigger stack.

I can say that a single configuration constant in the smaller stack's runtime appears to have been binding on multiple recurring failure modes, and changing it produced an observable improvement.

I cannot say that the bigger stack's tuning approach would generalize to a different smaller stack, a different product domain, or a different rubric. I cannot say whether the smaller stack would eventually have reached harness-modification parity with the bigger stack given more time. I cannot say which of the rules added during the window are still binding at the end of the window. I cannot say whether the cost ratio I observed (the smaller stack's model is cheap, the bigger stack's tuning is roughly an order of magnitude more expensive per iteration) is intrinsic to the setup or an artifact of this particular tuner's strategy.

I want to be explicit about what would have to be different for this to be more than a single field report. Multiple trajectories with different smaller stacks would help. An independent rater applying the rubric, instead of the tuner playing both roles, would help. Running the same setup with the tuner role frozen (no strategy adjustments) would help isolate which of the gains came from harness modifications versus from tuner adaptation. A larger primitive set on the smaller stack at the start, to see whether parity at primitive-modification is reachable when the starting gap is smaller, would help. I do not expect to do any of these myself in the near term. I am interested if anyone else does.

## Time and cost

About 23 hours of wall-clock from first sweep to last. About 10 hours of that had observable active work. The rest was inter-phase silences and overnight gaps.

The smaller stack ran about 46 iterations across the window. Token cost on those iterations, extrapolated from one iteration's telemetry, comes to roughly $2.30 total. That is the smaller stack at small-model rates.

The bigger stack's token cost is harder to estimate because I did not have its telemetry. I am budgeting it from what I could see of its outputs (commit messages, planning entries) and what I would expect for its inputs (context, diffs, event streams). My rough estimate is on the order of $40 to $50 for the tuner role across the window, plus another $10 or so for ad-hoc cleanup commits, plus a few dollars for the two times the tuner played the judge role. Total system cost in the range of $55 to $70, of which the smaller stack is a small fraction.

The order-of-magnitude conclusion (the bigger stack dominates total cost) feels robust to whatever uncertainty is in my estimates. The precise dollar figure is not. I would not put weight on the specific numbers without telemetered measurement.

For context: this trajectory cost something like the price of a moderate code-review session with a senior engineer, for ten hours of active work on one product surface. Whether that is economical depends on whether the smaller stack, once tuned, can run unsupervised on similar tasks. That question this trajectory does not answer.

## If you are trying something similar

These are suggestions, not recommendations. I would want to see them confirmed in other settings before treating them as advice.

If you can, set up the external scoring layer first and run it on a fixed cadence. In this trajectory, the rubric verdict broke a tuner strategy that probably would have continued otherwise. Without external scoring, I do not know what the tuner would have done.

When failure modes accumulate, consider auditing the substrate before reaching for more rules. The single change that mattered most in this trajectory was a configuration constant. I do not think it was unique in being substrate-driven; it was just the one the rubric forced the tuner to look at.

Build the tuner's verification on artifacts (working-tree state, build output, tool logs), not on the smaller stack's prose self-reports. The smaller stack's self-reports were wrong often enough that any pipeline that trusts them will accumulate error.

When designing rules for the smaller stack, prefer ones whose violation it can check by looking at its current output. Rules that require it to remember something across iterations seemed less reliable in this trajectory.

Be honest in your accounting about which stack is doing the work. If the bigger stack is doing most of the smaller stack's harness modifications, that is a tuning loop, not a self-evolution loop. The framing affects what you can claim about the smaller stack's standalone capability.

Be honest about the observer. If you have someone watching, they probably perturb the experiment. I did.

## Closing

The framing I would land on for what I watched is something like: an experiment in cross-stack harness tuning, where a bigger agent stack tried to bootstrap a smaller one toward harness-modification parity. The smaller stack's product-code output improved during the window. The smaller stack did not reach the bootstrap goal during the window; it could not, by the end, modify its own harness at quality the bigger stack would accept without rescue. Whether more time would have closed that gap is open.

I do not think I can say much more than that from one trajectory. If anyone reading this is running similar setups and wants to compare notes, especially with measurement methods that are less leaky than mine, I would be interested. The bootstrap question (can a smaller agent stack reach harness-modification parity with a bigger one through repeated tuning) seems like a question worth answering. I just do not think this trajectory answers it.

