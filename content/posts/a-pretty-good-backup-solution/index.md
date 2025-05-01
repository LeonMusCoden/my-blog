---
title: 'A pretty good backup solution'
date: '2025-05-01'
draft: false
categories:
  - default
tags:
  - selfhosting
---

"Data doesn't exist unless it exists in at least three places." 

This wisdom especially applies to all your selfhosted data. Often as a homelab grows, so does the importance of the data that's stored within. At least for me that holds true. So, how do I make sure my data is safe?


## Requirements

I have the following requirements for my backup solution:

1. Even if my whole homelab falls apart, I must still have a copy of my data
2. Rollback should be quick even if one copy is down
3. I can cherry-pick files to restore

There are only two exceptions to these requirements:
- *Jellyfin media*: Movies and shows don't need to be backed up as I can simply download them again from the high seas.
- *Immich photos*: These must follow the first and third requirements but don't need quick rollback capability.

## The 3-2-1 Backup Strategy

After researching various approaches, I decided to implement the classic 3-2-1 backup strategy, which stands for:

- 3 copies of your data
- On 2 different media types
- With 1 copy being offsite

### Implementation

Here's how I've implemented this strategy in my homelab:

#### Primary Storage (Copy 1)

All my data is consolidated on a TrueNAS server. Critical data that needs to be backed up (except photos) is stored on SSDs. I've set up regular snapshots that provide point-in-time recovery options directly on the primary storage.

#### Secondary Local Backup (Copy 2)

These snapshots get automatically replicated to a hard drive on another server within my network. This provides me with a second copy on a different media type and different hardware.

#### Offsite Backup (Copy 3)

Finally, I backup everything to Backblaze B2 - fully encrypted, of course. This includes my photo collection from Immich. This satisfies the "offsite" requirement of the 3-2-1 strategy.

## Results and Costs

This setup works very well for my needs. If I need to roll something back, I have multiple options:

- For quick recovery: I can use the local snapshots on the TrueNAS server
- If the primary storage is down: I can rely on the snapshots on the secondary server
- In case of catastrophic failure: All important data is also in the cloud

The solution is also surprisingly cost-effective. The backup to Backblaze only costs me about $3/month because I have just over 1TB of data to preserve.

## Future Plans

In the future, I might replace Backblaze with a backup to my friend's homelab - I'm still waiting for him to upgrade the storage in his NAS. This would bring my costs to 0.

