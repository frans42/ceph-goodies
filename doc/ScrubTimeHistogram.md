[WIP]

# Inspecting scrub time histograms

Use the script [`pool-scrub-report`](../scripts/pool-scrub-report) to print a scrub-time histogram for a pool. Usage is as in

```
pool-scrub-report MyPool
```

and the output looks like this:

```
Scrub info for pool sr-rbd-data-one-hdd (id=11): using cache (RTTL=4:16)

Scrub report:
  11%     119 PGs not scrubbed since  1 intervals (  6h)
  21%     102 PGs not scrubbed since  2 intervals ( 12h)
  33%     123 PGs not scrubbed since  3 intervals ( 18h)
  42%      91 PGs not scrubbed since  4 intervals ( 24h)
  52%     103 PGs not scrubbed since  5 intervals ( 30h)
  62%     107 PGs not scrubbed since  6 intervals ( 36h)
  70%      72 PGs not scrubbed since  7 intervals ( 42h)
  77%      76 PGs not scrubbed since  8 intervals ( 48h)
  87%     101 PGs not scrubbed since  9 intervals ( 54h) [9 idle]
  93%      61 PGs not scrubbed since 10 intervals ( 60h) [7 idle] [1 scrubbing+deep]
  96%      37 PGs not scrubbed since 11 intervals ( 66h) [3 idle]
  99%      25 PGs not scrubbed since 12 intervals ( 72h) [2 idle]
  99%       6 PGs not scrubbed since 13 intervals ( 78h) [1 scrubbing+deep]
 100%       1 PGs not scrubbed since 14 intervals ( 84h)
         1024 PGs, EST=2.37d (2.33d), 0 scrubbing, 21 (9.1%) idle, 0 unclean.

Deep-scrub report:
  10%     109 PGs not deep-scrubbed since  1 intervals ( 24h)
  21%     108 PGs not deep-scrubbed since  2 intervals ( 48h)
  31%     103 PGs not deep-scrubbed since  3 intervals ( 72h)
  39%      89 PGs not deep-scrubbed since  4 intervals ( 96h)
  46%      72 PGs not deep-scrubbed since  5 intervals (120h)
  53%      68 PGs not deep-scrubbed since  6 intervals (144h)
  58%      53 PGs not deep-scrubbed since  7 intervals (168h)
  61%      31 PGs not deep-scrubbed since  8 intervals (192h)
  65%      38 PGs not deep-scrubbed since  9 intervals (216h)
  68%      26 PGs not deep-scrubbed since 10 intervals (240h)
  70%      23 PGs not deep-scrubbed since 11 intervals (264h)
  76%      59 PGs not deep-scrubbed since 12 intervals (288h)
  80%      47 PGs not deep-scrubbed since 13 intervals (312h)
  87%      70 PGs not deep-scrubbed since 14 intervals (336h)
  94%      73 PGs not deep-scrubbed since 15 intervals (360h)
  99%      45 PGs not deep-scrubbed since 16 intervals (384h)
 100%      10 PGs not deep-scrubbed since 17 intervals (408h) 2 scrubbing+deep
         1024 PGs, EDST=14.98d, 2 scrubbing+deep, 0 unclean.

sr-rbd-data-one-hdd  scrub_min_interval=48h  (8i/80%/102PGs÷i)
sr-rbd-data-one-hdd  scrub_max_interval=168h  (7d)
sr-rbd-data-one-hdd  deep_scrub_interval=336h  (14d/~92%/~67PGs÷d)
osd.146  osd_scrub_interval_randomize_ratio=0.500000  scrubs start after: 48h..72h
osd.146  osd_deep_scrub_randomize_ratio=0.000000
osd.146  osd_max_scrubs=1
osd.146  osd_scrub_backoff_ratio=0.500000
mon.ceph-01  mon_warn_pg_not_scrubbed_ratio=0.500000  warn: 10.5d (42.0i)
mon.ceph-01  mon_warn_pg_not_deep_scrubbed_ratio=0.750000  warn: 24.5d
```

PGs are allocated to buckets of width 6h for scrub- and 24h for deep-scrub times according to their respective (deep-)scrub stamps. The first column gives the cumulative percentage of PGs up to and including this bucket and corresponds to the empirical probability function. The second column is the number of PGs in a bucket and is a representation of the empirical probability distribution function.

For PGs with scrub times larger than scrub_min_interval and deep-scrub times larger than deep_scrub_interval additional status information is printed at the end giving an indication of how many PGs might be scrubbing and why PGs are not scrubbing. For buckets under "Deep-scrub report" that have no more than 5 PGs in it more detail is printed: PG IDs that are idle+eligible and how many PGs are busy. For these small buckets the count should add up to the number of PGs in that bucket.

For the pool above deep-scrub times for a PG are between 25-45 minutes with an average of about 30min. The [formula for the recommended deep-scrub interval](RecommendationsForScrub.md#adjust-deep-scrub-time-for-pools-on-hdds) suggests 10.7 days for deep_scrub_interval and using scrub_interval=2 days and deep_scrub_interval=14 days results in an unproblematic deep-scrub load with a reasonable scrub stamp distribution over the scrub intervals.

## Status info and stats shown with the histograms

Besides the time stamp histograms the script outputs additional information that is intended to help with tuning and interpreting the effect of important parameters and also helps with discovering and fixing bugs in the script.

The line

```
1024 PGs, EST=2.37d (2.33d), 0 scrubbing, 21 (9.1%) idle, 0 unclean.
```

just below the scrub stamp histogram shows the total PG count, the empirical/estimated scrub time (EST) in days, the theoretical expected scrub time in parentheses, the number of PGs scrubbing, the number and percentage of idle PGs and how many PGs are considered unclean by the script. For example, "remapped" and "backfilling" PGs are considered clean, which is relevant if `osd_scrub_during_recovery=true`.

The EST and its theoretical value are the most important indicators of problems. If both numbers are almost identical, (deep-)scrubbing works fine. If EST is consistently larger than the theoretical value, the OSDs can't keep up with (deep-)scrubs and tuning of scrub parameters is required. It is quite normal that EST grows periodically. However, it should drop back to values close to the theoretical value within a full deep-scrub cycle.

The line

```
1024 PGs, EDST=14.98d, 2 scrubbing+deep, 0 unclean.
```

just below the deep-scrub stamp histogram shows the total PG count, the empirical/estimated deep-scrub time (EDST) in days, the number of PGs deep-scrubbing and how many PGs are considered unclean by the script.

The block

```
sr-rbd-data-one-hdd  scrub_min_interval=48h  (8i/80%/102PGs÷i)
sr-rbd-data-one-hdd  scrub_max_interval=168h  (7d)
sr-rbd-data-one-hdd  deep_scrub_interval=336h  (14d/~92%/~67PGs÷d)
```

shows the current per-pool scrub interval settings in hours and their conversion to intervals (unit i) or days (unit d). The second number shows the total expected percentage of PGs that will reside in the buckets with age less or equal to (deep_)scrub_interval if all PGs are evenly distributed over all buckets according to their respective relative capacity. We call this distribution the *equilibrium distribution*. For deep-scrub the number is prefixed with "~", because the exact formula is unknown at the time of writing. This might be updated in the future.

The relative capacity of a bucket depends mainly on the setting `osd_scrub_interval_randomize_ratio` and our calculation for bucket capacity for deep-scrub is only valid for `osd_deep_scrub_randomize_ratio=0`, which we recommend to use in any case.

The last number shows how many PGs need to be (deep-)scrubbed per interval (day) in order to approach the equilibrium distribution. If not enough PGs can be (deep-)scrubbed per interval (day), scrubbing will be unstable with bursts of thundering herds of deep-scrubs and "PGs not (deep-)scrubbed in time" warnings are to be expected.

The block

```
osd.146  osd_scrub_interval_randomize_ratio=0.500000  scrubs start after: 48h..72h
osd.146  osd_deep_scrub_randomize_ratio=0.000000
osd.146  osd_max_scrubs=1
osd.146  osd_scrub_backoff_ratio=0.500000
```

shows relevant per-OSD settings and their interpretation. The OSD selected for reading these parameters is the first primary OSD of the first PG in the pool reported by "pg dump". This selection is not guaranteed to be stable and can vary from invocation to invocation.

The parameter `osd_scrub_interval_randomize_ratio` randomizes the start of scrubs to "smear out" scrub time stamps evenly over time. The time window when scrubs will start is shown in hours. The parameters `osd_deep_scrub_randomize_ratio` and `osd_max_scrubs` are shown for confirming they are set to recommended values. Using other values can have very bad results, which makes it quite important to check these parameters.

In this particular example, we use a non-default value for `osd_scrub_backoff_ratio`. This parameter has less influence than one might conclude from the documentation shown by `ceph config help osd_scrub_backoff_ratio`. Its intention is to balance aggressiveness of scrub scheduling with race conditions occurring when trying to allocate scrub reservations and it seems to have a rather flat objective function with a larger interval of equally good values. For our HDD pools using 0.5 led to a somewhat better scrub slot utilization indicated by a somewhat lower idle% value below the scrub time histogram. For pools that are *not* constantly deep-scrubbing, this parameter can be ignored (left at default).

The block

```
mon.ceph-01  mon_warn_pg_not_scrubbed_ratio=0.500000  warn: 10.5d (42.0i)
mon.ceph-01  mon_warn_pg_not_deep_scrubbed_ratio=0.750000  warn: 24.5d
```

shows monitor settings controlling the appearance of scrub-related warnings. The values in intervals and days show the resulting (deep-)scrub stamp ages after which warnings will be shown.

---
Next: [The scrub slot paradox.](ScrubSlotParadox.md)
Back: [Terms and definitions.](ScrubTerms.md)
Start: [Scrub tuning guide.](TuningScrub.md)
---

# Historic scrub time histograms

[WIP]

In this section we plan to present different histograms obtained during our experiments with different config settings to illustrate the effects that certain settings had and how we arrived at our recommendations.

---
Next: [The scrub slot paradox.](ScrubSlotParadox.md)
Back: [Terms and definitions.](ScrubTerms.md)
Start: [Scrub tuning guide.](TuningScrub.md)