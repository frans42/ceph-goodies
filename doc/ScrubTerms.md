# Terminology

## Idle and eligible PGs

We call a PG *idle* if it is active+clean and no OSD that is a member of this PG is scrubbing or scrubbing+deep. We call a PG *eligible* for (deep-)scrub if its (deep-)scrub age has passed the (deep-)scrub threshold. PGs that are eligible+idle can be (deep-)scrubbed at any time. This explains the mystery why a "`ceph pg deep-scrub x.y`" often does not cause a PG to deep-scrub (immediately). The command forces the PG to become eligible, but (deep-)scrub start has to wait until it also becomes idle.

---
Next: [Inspecting scrub time histograms.](ScrubTimeHistogram.md)
Back: [Recommendations for scrub settings.](RecommendationsForScrub.md)
Start: [Scrub tuning guide.](TuningScrub.md)