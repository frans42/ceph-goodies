[WIP]

# Inspecting scrub time histograms

Use the script [`pool-scrub-report`](../scripts/pool-scrub-report) to print a scrub-time histogram for a pool. Usage is as in

```
pool-scrub-report MyPool
```

and the output looks like this:

```
Scrub info for pool sr-rbd-data-one-hdd (id=11): dumped pgs

Scrub report:
  12%     133 PGs not scrubbed since  1 intervals (  6h)
  21%      90 PGs not scrubbed since  2 intervals ( 12h)
  30%      88 PGs not scrubbed since  3 intervals ( 18h)
  40%     103 PGs not scrubbed since  4 intervals ( 24h)
  50%     107 PGs not scrubbed since  5 intervals ( 30h)
  58%      82 PGs not scrubbed since  6 intervals ( 36h)
  68%      95 PGs not scrubbed since  7 intervals ( 42h)
  78%     108 PGs not scrubbed since  8 intervals ( 48h)
  87%      93 PGs not scrubbed since  9 intervals ( 54h) [30 idle]
  94%      65 PGs not scrubbed since 10 intervals ( 60h) [20 idle] [1 scrubbing+deep]
  98%      41 PGs not scrubbed since 11 intervals ( 66h) [18 idle]
  99%      17 PGs not scrubbed since 12 intervals ( 72h) [4 idle]
  99%       1 PGs not scrubbed since 13 intervals ( 78h)
 100%       1 PGs not scrubbed since 14 intervals ( 84h)
         1024 PGs, EST=2.36d (2.33d), 0 scrubbing, 72 idle, 0 unclean.

Deep-scrub report:
   8%      82 PGs not deep-scrubbed since  1 intervals ( 24h)
  13%      53 PGs not deep-scrubbed since  2 intervals ( 48h)
  17%      45 PGs not deep-scrubbed since  3 intervals ( 72h)
  19%      16 PGs not deep-scrubbed since  4 intervals ( 96h)
  21%      20 PGs not deep-scrubbed since  5 intervals (120h)
  22%      18 PGs not deep-scrubbed since  6 intervals (144h)
  25%      29 PGs not deep-scrubbed since  7 intervals (168h)
  31%      58 PGs not deep-scrubbed since  8 intervals (192h)
  37%      59 PGs not deep-scrubbed since  9 intervals (216h)
  44%      79 PGs not deep-scrubbed since 10 intervals (240h)
  53%      88 PGs not deep-scrubbed since 11 intervals (264h)
  63%     100 PGs not deep-scrubbed since 12 intervals (288h)
  73%     106 PGs not deep-scrubbed since 13 intervals (312h)
  86%     132 PGs not deep-scrubbed since 14 intervals (336h)
  96%     101 PGs not deep-scrubbed since 15 intervals (360h) 1 scrubbing+deep
  99%      36 PGs not deep-scrubbed since 16 intervals (384h)
 100%       2 PGs not deep-scrubbed since 17 intervals (408h) [2 busy]
         1024 PGs, EDST=14.72d, 1 scrubbing+deep, 0 unclean.

sr-rbd-data-one-hdd  scrub_min_interval=48h  (8i/80%/102PGs÷i)
sr-rbd-data-one-hdd  scrub_max_interval=168h  (7d)
sr-rbd-data-one-hdd  deep_scrub_interval=336h  (14d/~92%/~67PGs÷d)
osd.221  osd_scrub_interval_randomize_ratio=0.500000  scrubs start after: 48h..72h
osd.221  osd_deep_scrub_randomize_ratio=0.000000
osd.221  osd_max_scrubs=1
osd.221  osd_scrub_backoff_ratio=0.500000
mon.ceph-01  mon_warn_pg_not_scrubbed_ratio=0.500000  warn: 10.5d (42.0i)
mon.ceph-01  mon_warn_pg_not_deep_scrubbed_ratio=0.750000  warn: 24.5d
```

PGs are allocated to buckets of width 6h for scrub- and 24h for deep-scrub times according to their respective (deep-)scrub stamps. The first column gives the cumulative percentage of PGs up to and including this bucket and corresponds to the empirical probability function. The second column is the number of PGs in a bucket and the column is a representation of the empirical probability distribution function.

For PGs with scrub times larger than scrub_min_interval and deep-scrub times larger than deep_scrub_interval additional status information is printed at the end giving an indication of how many PGs might be scrubbing and why PGs are not scrubbing.
