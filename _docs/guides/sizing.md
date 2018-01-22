---
title: Sizing Deployment
category: Guides
order: 3
---

> Ported from [Confluence](https://dominodatalab.atlassian.net/wiki/spaces/CS/pages/21934733/Dispatcher+Sizing)

Ozzy's August 2017 thoughts on Dispatcher sizing:

tl;dr - Pay attention to dispatcher load average.

- A co-located dispatcher with a 4GB heap is good for about 40 concurrent runs in the best case. Load average should be < 1.
- If it's creeping into double-digits your well past time to resize.
- We have a domino.java_max_heap setting in the deployer with dispatcher and frontend keys. Don't forget to adjust it when resizing the machine.

Over time, I've come up with a few anecdotal rules about sizing based on observed performance in growing installs.

1. Dispatcher performance /  load is largely bound by the number of concurrent runs. Every additional run contributes somewhere in the neighborhood of 100 calls per minute to the dispatcher. The dispatcher can handle somewhere in the neighborhood of 15-20 runs per core (I suspect this varies based on the latency of Mongo). Over that number the load average will start to spike and eventually peg in the 100s as some number of these calls presumably go unserved and timeout.

   A healthy dispatcher should see a load average in the very low single digits at most. This is important to keep an eye on as it's a case where things can break bad very quickly and as the problem isn't likely to reflected in the more common view of utilization.

2. Dispatcher load can be spiky outside of concurrent runs since it handles the Search ES indexing and has the fluentd aggregator co-located alongside. Re-indexing or indexing large numbers of run-generated files will produce high utilization spikes and contribute to existing load average issues.

3. Since the dispatcher is the fluent aggregator, large amounts of stdout can exacerbate #1 and #2.
