WIP: read with care.

# Tuning the scrub machine

*Note: while the discussion below focuses mainly on HDDs due to their low IOPS performance per TB, we also address solid state storage and give recommendations.*

## General recommendation

In our investigation we found that the following two recommendations apply in general for all drive types. These are the two main settings to reduce sustained deep-scrub load while improving scrub time and eliminating long tails in the scrub time histogram. It is assumed that all other scrub settings are at default. If these two settings together with adjusting the scrub intervals are not sufficient, more fine-grained tuning is required as described further down.

### Always set osd_deep_scrub_randomize_ratio=0

Command: `ceph config set osd osd_deep_scrub_randomize_ratio 0`

The parameter osd_deep_scrub_randomize_ratio was introduced to prevent [thundering herds of deep-scrubs](https://lore.kernel.org/all/CABZ+qq=7izv5o-5ACygqZyr=ho58nLoKb7XmRKT2yyTqKFwrZQ@mail.gmail.com/). While it does have this effect, it also has serious side effects that are detrimental to overall performance:

1. It leads to a large amount of premature deep-scrubs, which means reduced user-experience due to unnecessarily deep-scrubbing too early too often (highly increased sustained deep-scrub load).
2. It does not lead to an evenly distributed scrub time histogram, nor does it allow to configure something like "scrub every 2-3 days, deep-scrub every 14-21 days".
3. It leads to long tails in the scrub time distribution with the "PG not (deep-)scrubbed in time" warnings piling up and never disappearing again.

Before setting this parameter to 0, [inspect the scrub time histogram](#inspecting-scrub-time-histograms) for your pool and consider gradually reducing this parameter to 0 over time. We set it to 0 right away and it improved the situation for us immediately.

More details including example histograms are included in the detailed discussion below.

### Never set osd_max_scrubs>1

Command: `ceph config set osd osd_max_scrubs 1` or `ceph config rm osd osd_max_scrubs`

Setting osd_max_scrubs=2 on a HDD pool was a mistake we made while trying to deal with "PGs not (deep-)scrubbed in time" warnings. The net effect was actually the opposite. After the number of warnings continued increasing, we investigated the OSD logs and found the following:

- osd_max_scrubs=1 => deep-scrub time ca. 30min per 1 PG
- osd_max_scrubs=2 => deep-scrub time ca. 70min per 2 PGs

Deep-scrubbing 2 PGs simultaneously took 70min while deep-scrubbing 2PGs sequentially took only 60min. Conclusion: HDDs cannot deal effectively with more than 1 deep-scrub at a time and for pools on SSDs we have never seen it necessary to deep-scrub more than 1 PG at a time. Do not set osd_max_scrubs>1, it will make a bad situation worse.

### Adjust deep scrub time for pools on HDDs

For HDD pools a deep-scrub every week is often too frequent considering drive performance. Good alternatives are

1. scrub_min_interval=2 days, deep_scrub_interval=2 weeks,
2. scrub_min_interval=3 days, deep_scrub_interval=2 weeks or
3. scrub_min_interval=3 days, deep_scrub_interval=3 weeks

depending on drive size and performance. The default ratio of scrub_min_interval:deep_scrub_interval=1:7 is a good choice (option 1 and 3) and should be used unless a different ratio is really required (like option 2).

The parameters need to be set in seconds, use the following values:

| scrub interval | seconds | | deep scrub interval | seconds |
|       ---      |  ---:  |---|       ---          |   ---:  |
| 1 day  |  86400 | | 1 week  |  604800 |
| 2 days | 172800 | | 2 weeks | 1209600 |
| 3 days | 259200 | | 3 weeks | 1814400 |

Use the pool settings for these values as in

```
ceph osd pool set MyPool scrub_min_interval  172800
ceph osd pool set MyPool deep_scrub_interval 1209600
```

# Advanced tuning of the scrub machine

## Introduction

The subsequent investigation was the result of trying to fix issues with PGs not being deep scrubbed in time. After blindly trying a few recommendations provided by the ceph user list making matters worse, we decided to look under the hood and find out why we had such problems and what the actual result of certain tuning parameters is.

Threads and posts related to problems addressed and problems observed are (search for thread title if the link is dead, some threads might have moved to archive):

- [Thundering herd of deep-scrubs](https://lore.kernel.org/all/CABZ+qq=7izv5o-5ACygqZyr=ho58nLoKb7XmRKT2yyTqKFwrZQ@mail.gmail.com/)
- [increasing number of (deep) scrubs](https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/thread/NHOHZLVQ3CKM7P7XJWGVXZUXY24ZE7RK)
- [How to configure something like osd_deep_scrub_min_interval?](https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/thread/YUHWQCDAKP5MPU6ODTXUSKT7RVPERBJF)
- [Ceph and Deep-Scrubs](https://silvenga.com/ceph-and-deep-scrubs/)

With HDDs becoming larger while the IOPS performance per disk stays essentially constant, the available IOPS per TB are decreasing with disk size. For disks of sizes >=5-10TB this leads to the problem that it becomes more and more difficult to perform admin operations like scrub and deep-scrub alongside user-IO.

With the current default scrub settings for ceph this leads to "PGs not (deep-)scrubbed in time" warnings that start piling up and scrub seems never to be able to catch up again. For large HDDs one needs to look into the scrub parameters together with what performance HDDs can actually provide to find a balance between user experience and (deep-)scrub times.

A side conclusion of the observations reported here is that there is an upper limit of HDD size for which HDDs become essentially useless for non-archival (or even any?) storage systems. The hard upper limit is reached when the sustained IOPS performance/TB is too low to complete (deep-)scrubbing with reasonable scrub-times at all. The soft limit is when remaining IOPS per TB performance is so low that users consider the storage unusable. Our experience indicates that this soft limit is reached somewhere around 5-10 IOPS per TB, which are drives of size 16-32TB capacity.

As a consequence, we limit drive sizes to <=16TB and aim for no more than 10TB/drive utilization. This is the first step to ensure that a HDD pool can serve reasonable IOPS while (deep-)scrubbing within acceptable intervals.

All this, of course, depends on actual workload and will be different from cluster to cluster. To add some background, we are here mainly concerned with an 850+ HDD-OSD pool with EC8+3 profile used as a data pool in a ceph fs. Usable capacity is about 9PB and used is 5.3PB. We have a large percentage cold data and a thin layer of between 50-100TB hot data. User IO hits the hot data while scrubbing must go through everything.

This pool has approximately an aggregated IOPS budget of 5800, which sounds little but is good enough for our use case of a bulk data store. In the future we plan a hybrid OSD config with dm-cache layer for the hot data on solid state storage. One of the major issues is that we have a larger percentage of small files and also a lot of hard links on the file system, which are a pain for both, deep-scrub and recovery.



## Goal

With our tuning recommendations we aim at configuring (dep-)scrub such that:

- PGs scrub every A-B days
- PGs deep-scrub every C-D days
- no PGs (deep-)scrub before (C)A days, that is, no premature (deep-)scrubbing
- PGs with (deep-)scrub ages <(C)A should be evenly distributed in the scrub time histogram
- (deep-scrub) load under normal conditions should be low enough such that the scrub machine has sufficient reserves to handle exceptional situations causing (deep-)scrub to pause, for example, rebalancing after adding OSDs or periods of high load on the servers; (deep-)scrub should catch up after a reasonable amount of time

With the current implementation of the ceph scrub machine and parameters available for tuning we were able to find configurations that approximate these goals sufficiently well for our purposes. In most cases, the recommendations above will do.

## Terminology and relevant parameters

We call a PG *idle* if it is active+clean and no OSD that is a member of this PG is scrubbing or scrubbing+deep. We call a PG *eligible* for (deep-)scrub if its (deep-)scrub age has passed the (deep-)scrub threshold. PGs that are eligible+idle can be (deep-)scrubbed at any time. This explains the mystery why a "`ceph pg deep-scrub x.y`" often does not cause a PG to deep-scrub (immediately). Deep-scrub has to wait until the PGs state becomes eligible+idle.


sr-rbd-data-one-hdd  scrub_min_interval=48h  (8i/80%/102PGs÷i)
sr-rbd-data-one-hdd  scrub_max_interval=168h  (7d)
sr-rbd-data-one-hdd  deep_scrub_interval=336h  (14d/~92%/~67PGs÷d)
osd.221  osd_scrub_interval_randomize_ratio=0.500000  scrubs start after: 48h..72h
osd.221  osd_deep_scrub_randomize_ratio=0.000000
osd.221  osd_max_scrubs=1
osd.221  osd_scrub_backoff_ratio=0.500000


## Inspecting scrub time histograms

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

## Towards an evenly-distributed scrub-time histogram

## Recommendations for scrub parameters
