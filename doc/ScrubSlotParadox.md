# The Scrub Slot Paradox

In the thread [increasing number of (deep) scrubs](https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/thread/NHOHZLVQ3CKM7P7XJWGVXZUXY24ZE7RK) ([thread in archive](https://www.spinics.net/lists/ceph-users/msg75292.html)) we were asking why deep scrubs are lagging behind while only about 232/852=27% of OSDs are scrubbing and it would be easy to catch up if a higher percentage of OSDs would deep scrub as well.

An alternative formulation of this question is: we have 12 servers with 71 HDD OSDs, which back 2 pools with 8+2 (2048 PGs) and 8+3 (8192) EC profiles and failure domain host. Naively, one could say that it should be possible to scrub 71 PGs simultaneously instead of the 20-22 PGs we observe. Surely, if there are no more than 22 OSDs busy per host it should be possible to scrub a few PGs more at the same time, because there are still 49 OSDs per host idle.

The surprising answer is: actually, no (and a little bit yes; see comment at the end).

Lets look at a simplified situation for which we can calculate statistics: take an 8+3 EC pool on 11 hosts with 71 OSDs each, a setting where every PG has exactly 1 OSD in each host as a member. If all combinations of disks per host are equally likely and selected randomly for each of our PGs, how many PGs can we expect to exist on a subset of 71-S OSDs per host? The answer is given by the ratio of the [number of permutations](https://en.wikipedia.org/wiki/Permutation#Permutations_with_repetition) of disks in the hosts minus the number of PGs scrubbing and the total number of permutations times the expected number of eligible PGs:

    N=#P(71-S,11)/#P(71,11)*#PGs_eligible,

where `#P(X,Y)=X^Y` is the cardinal number of the set of Y-tuples of X elements. N is the expected value of #PGs that exist on the remaining 71-S OSDs after removing the OSDs of the S PGs that are scrubbing. If `N>=1` we can expect that at least 1 additional PG can be scrubbed. Note that the formula simplifies to

    N=0.1*8192*((71-S)/71)^11,

where the factor 0.1 comes from the assumption that at equilibrium 10% of PGs are eligible for scrubbing (for well-performing OSDs this number is usually close to 0). Let's look at some numbers for increasing values of S (the row N states the floor(N) to have an integer number):

 S |  0  |  5  | 10  | 15 | 20 | 25 | 30 |
---|:---:|:---:|:---:|:--:|:--:|:--:|:--:|
 N | 819 | 366 | 154 | 60 | 21 | 6  | 1  |

The surprising result is that even with 10% of PGs being eligible for scrubbing no more than 31 PGs can be expected to scrub simultaneously, leaving 40 or more OSDs per host idle at all times! Furthermore, due to the large exponent of 11=8+3, increasing the number of PGs of the pools has a limited effect only. It is likely to increase utilization of the number of _scrub slots_ `SL` (see definition below) while it has limited effect on the number of scrub slots itself.

Interestingly, this result seems invariant if the number of hosts is increased. This only adds a common factor to each of the permutation terms, which then divide out to 1. Furthermore, from

    N=0.1*8192*((71-S)/71)^11
     =0.1*8192*((71*11-S*11)/71*11)^11
     =0.1*8192*((781-S*11)/781)^11
     =E*#PGs*((#OSDs-S*REP)/#OSDs)^REP                 (1)
     =#PGs_eligible*(#OSDs_idle/#OSDs)^REP

we can express the key result in terms of #OSDs and replication factor alone, the number of hosts seems not to matter at all. The main parameter is the replication factor that enters as the exponent into the ratio of idle OSDs and total number of OSDs. The larger the replication factor of a pool, the fewer scrub slots will be available for the same number of PGs.

Formula (1) can be rewritten to obtain a generally useful expression for the number of scrub slots for any given pool. Keeping all parameters fixed and varying only S, the number SL of scrub slots has the property (N(SL-1)>=1 && N(SL)<1), meaning if there are SL PGs scrubbing we expect that on average less than 1 PGs exist on the remaining idle OSDs. In other words, on average no more than SL PGs will ever scrub at the same time.

Using this observation and assuming `E*#PGs>=1`, we can solve

    N(SL)=1=E*#PGs*((#OSDs-(SL-1)*REP)/#OSDs)^REP

for SL and obtain

    SL=1+(#OSDs/REP)*(1-(E*#PGs)^(-1/REP)).

Here:

- `SL`: number of scrub slots
- `#OSDs`: number of OSDs
- `REP`: replication factor for pool
- `E`: percentage of eligible PGs in equilibrium

If multiple pools live on the same OSDs, the calculation becomes more complicated because the scrub slots will be shared between pools with possibly different replication factors. If one substitutes the total number of PGs over all pools, then using the smallest replication factor will give an upper and using the largest replication factor a lower bound for SL.

With the example above we obtain `SL=1+(781/11)*(1-(0.1*8192)^(-1/11))=33` (taking the floor as usual to obtain integers). If we add the extra host, we get `1+(852/11)*(1-(0.1*8192)^(-1/11))=36`. Just for fun, in the extreme case of E=1 (`scrub_min_interval=0`, all PGs are eligible all the time) we get 40 (44 with the extra host)! That's not a lot more and marks the hard limit of how many scrub slots a pool has.

There is still quite a gap between the number 33 of scrub slots for pool 2 alone and the observation in the ceph-user post of no more than 20-22 PGs of pool 1 _and_ 2 scrubbing simultaneously. The reason for this discrepancy is a low scrub slot utilization caused by [hanging scrub reservations](StuckScrubReservations.md).
