---
title: "Valorant’s anti-cheat just made some expensive cheat hardware useless"
source: "https://www.digitaltrends.com/gaming/valorants-anti-cheat-just-made-some-expensive-cheat-hardware-useless/"
author:
  - "[[Sudhanshu Kumar Mangalam]]"
published: 2026-05-23
created: 2026-05-23
description: "Riot has clarified how Vanguard’s latest update targets DMA-based cheat hardware by tightening hardware-level memory protections. The crackdown may stop advanced cheaters, but it is also raising questions about anti-cheat overreach."
tags:
  - "clippings"
---
[Riot Games](https://www.digitaltrends.com/gaming/riot-games-valorant-tactical-spin-on-overwatch/) ’ controversial Vanguard anti-cheat has drawn fresh attention after a new update reportedly made some expensive [Valorant](https://www.digitaltrends.com/gaming/what-is-flex-in-valorant/) cheat hardware unusable. Riot mocked hardware cheaters on X, saying “congrats to the owners of a brand new $6k paperweight,” after the update appeared to block DMA-based cheat setups that rely on costly external hardware.

The comment sparked debate around what Vanguard had actually done. Some posts framed the issue as [SSD](https://www.digitaltrends.com/computing/best-ssds/) damage, but Riot has clarified that Vanguard does not damage PC hardware or disable real SSDs. The update targets cheat hardware and firmware used to bypass Valorant’s anti-cheat systems.

See More

## What is DMA cheating?

DMA stands for Direct Memory Access. It is a normal hardware function that lets devices access system memory without routing every request through the CPU. In cheating setups, DMA can be abused to read game memory from outside the normal software layer.

Recommended Videos

A DMA device can be connected to a PC through PCIe, allowing cheaters to run tools such as radar, wallhacks, or ESP through a separate machine. These setups are often expensive and use special programmable hardware made to look like normal PC devices, making them harder to spot.

![Key art for the console release of Valorant.](https://www.digitaltrends.com/tachyon/2024/08/Valorant-Xbox.jpg?resize=2597%2C1461)

Riot Games

## What did Riot change?

The latest Vanguard update seems to target a loophole used by high-end cheating setups that depend on external hardware rather than conventional software. These devices impersonate trusted components like storage drives, allowing them to access game data while avoiding easy detection.

To counter that, Riot seems to be enforcing stricter use of IOMMU (Input-Output Memory Management Unit), a hardware-level memory protection system that controls what connected devices are allowed to access. By locking down those permissions, Vanguard can prevent suspicious external hardware from reading live game data, effectively cutting off the advantage DMA cheats are built around.

## Did Riot “brick” real SSDs?

[According to Riot](https://x.com/riotgames/status/2057879738350444726), Vanguard does not damage hardware, disable devices, or brick PCs, PC components, or software. The company says the affected devices are DMA cheat tools sold for Valorant cheating, not regular SSDs or PC parts. Some affected users have claimed they needed to reinstall Windows because their systems became unusable.

See More

Riot says the instability happens when IOMMU protections are enabled and a cheat setup continues trying to access protected memory. In that case, the system may generate hardware faults or become unstable. This is expected IOMMU behavior when a device tries to read memory it is no longer allowed to access.

## Why did this spark a trust debate

Community reaction has been split. Many players on X and Reddit mocked the affected cheaters, saying anyone spending thousands on DMA cheat hardware deserved to lose access to it.

<iframe src="https://embed.reddit.com/r/gaming/comments/1tki2q5/valorants_new_vanguard_update_seems_to_be/?embed=true&amp;ref_source=embed&amp;ref=share&amp;utm_medium=widgets&amp;utm_source=embedv2&amp;utm_term=23&amp;utm_name=post_embed&amp;embed_host_url=https%3A%2F%2Fwww.digitaltrends.com%2Fgaming%2Fvalorants-anti-cheat-just-made-some-expensive-cheat-hardware-useless%2F" width="640px" allowfullscreen="true" allow="clipboard-read; clipboard-write" height="240"></iframe>

Others were more uneasy about what this could mean for players. Some Reddit users questioned whether a [kernel-level anti-cheat](https://www.digitaltrends.com/gaming/riot-games-100000-valorant-vanguard/) should be able to block hardware at this level. Players are asking what would happen if legitimate hardware were wrongly flagged. Some worry that a false positive could block access to a genuine NVMe or SATA drive and force a Windows reinstall. Riot has denied that Vanguard disables normal PC devices or bricks PCs and PC components, but not everyone is convinced.

Stopping cheaters matters in a competitive game like Valorant, but kernel-level anti-cheats raise ethical questions because they sit deep inside a player’s system. When an anti-cheat can restrict hardware behavior, even for a valid reason, it creates a trust issue between the game maker and the player.