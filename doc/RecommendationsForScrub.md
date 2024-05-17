# General recommendation

In our investigation we found that the following two recommendations apply in general for all drive types. These are the two main settings to reduce sustained deep-scrub load while improving scrub time and eliminating long tails in the scrub time histogram. It is assumed that all other scrub settings are at default.

## Always set osd_deep_scrub_randomize_ratio=0

Command: `ceph config set osd osd_deep_scrub_randomize_ratio 0`

The parameter osd_deep_scrub_randomize_ratio was introduced together with osd_scrub_interval_randomize_ratio to prevent [thundering herds of deep-scrubs](https://lore.kernel.org/all/CABZ+qq=7izv5o-5ACygqZyr=ho58nLoKb7XmRKT2yyTqKFwrZQ@mail.gmail.com/). While it does have an additional effect, it also has serious side effects that are detrimental to overall performance:

1. It leads to a large amount of premature deep-scrubs, which means reduced user-experience due to unnecessarily deep-scrubbing too early too often (highly increased sustained deep-scrub load).
2. It does not lead to an evenly distributed scrub time histogram, nor does it allow to configure something like "scrub every 2-3 days, deep-scrub every 14-21 days".
3. It leads to long tails in the scrub time distribution with the "PG not (deep-)scrubbed in time" warnings piling up and never disappearing again.

Setting this parameter to 0 improved the situation for us immediately and the default value for osd_scrub_interval_randomize_ratio is actually sufficient for reasonably fast convergence of (deep-)scrub stamps to an equi-distributed [scrub stamp histogram](ScrubTimeHistogram.md).

## Never set osd_max_scrubs>1

Command: `ceph config set osd osd_max_scrubs 1` or `ceph config rm osd osd_max_scrubs`

Setting osd_max_scrubs=2 on a HDD pool was a mistake we made while trying to deal with "PGs not (deep-)scrubbed in time" warnings. The net effect was that the number of warnings continued increasing (a bit slower) while users where now also suffering from slow ops constantly. To understand what was going on, we investigated the OSD logs and found the following:

- osd_max_scrubs=1 => deep-scrub time ca. 30min per 1 PG
- osd_max_scrubs=2 => deep-scrub time ca. 70min per 2 PGs

Deep-scrubbing 2 PGs simultaneously took 70min while deep-scrubbing 2PGs sequentially took only 60min. Conclusion: HDDs cannot deal effectively with more than 1 deep-scrub at a time and for pools on SSDs we have never seen it necessary to deep-scrub more than 1 PG per OSD at a time. Do not set osd_max_scrubs>1, it will make a bad situation worse.

While the increased number of PGs deep-scrubbing simultaneously (it was more than double the number compared to using osd_max_scrubs=1) did help somewhat with the warnings, the gain was absolutely not worth the loss of user-io performance.

# PG_NUM and balancing PG distribution

Additional basic measures to improve scrubbing performance are a sufficiently large PG_NUM per pool and a balanced distribution of PGs across OSDs. The PG number per pool should be set as high as the OSD count allows and, if several pools share the same OSDs, should be balanced between pools such that the average deep-scrub times of PGs in different pools are as equal as possible. Specifically, if it is possible to reduce deep-scrub times to below 5 minutes, deep-scrubbing will perform very well.

# Additional recommendations for large OSDs and pools with wide EC-profiles

If the adjustments described above do not suffice to (deep-)scrub PGs in time one needs to take disk performance into account and calculate realistic deep scrub times.

## Adjust deep scrub time for pools on HDDs

For EC pools with high replication factor and living on large disks, mainly HDDs but potentially also large low-end SSDs, deep-scrubbing all PGs within a short time window can be challenging. To find out what deep-scrub times are realistic, one can estimate the time window as

    DSI = 2 * ADST_PG * #PGs / SL / 24

where

- `DSI`: `deep_scrub_interval` in days for pools living on the same disks
- `ADST_PG`: average deep-scrub time of a PG in hours (inspect OSD logs for this using commands like `grep "deep-scrub" /var/log/ceph/ceph-osd.ID.log`)
- `#PGs`: total number of PGs of pools living on the same disks
- `SL`: [number of scrub slots](ScrubSlotParadox.md), use lower bound for conservative estimate

The factor 2 accounts for [low scrub slot utilization](StuckScrubReservations.md) and dividing by 24 converts hours to days. Together with deep_scrub_interval one should also adjust scrub_min_interval. The default ratio of scrub_min_interval:deep_scrub_interval=1:7 is a good choice and should be used unless a different ratio is really required. These parameters need to be set in seconds and, if using full weeks for deep_scrub_interval, example choices are

| scrub interval | seconds | | deep scrub interval | seconds |
|       ---      |  ---:  |---|       ---          |   ---:  |
| 1 day  |  86400 | | 1 week  |  604800 |
| 2 days | 172800 | | 2 weeks | 1209600 |
| 3 days | 259200 | | 3 weeks | 1814400 |

We recommend to use the pool settings for these values as in

```
ceph osd pool set MyPool scrub_min_interval  172800
ceph osd pool set MyPool deep_scrub_interval 1209600
```

## Bump scrub throughput by clearing hanging scrub reservations

If the deep_scrub_interval obtained from the formula in the previous section is unacceptably high, running the [script bump_scrubs](../scripts/bump_scrubs) in a cron job allows to reduce the factor 2 accounting for [low scrub slot utilization](StuckScrubReservations.md). If this is still not sufficient, the only way forward is to use smaller and/or better performing disks.
