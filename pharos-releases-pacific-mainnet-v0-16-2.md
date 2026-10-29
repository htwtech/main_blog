---
title: 'Pharos Releases Pacific Mainnet v0.16.2 With Execution-Layer Stability Improvements'
slug: 'pharos-releases-pacific-mainnet-v0-16-2'
canonical_url: 'https://htw.tech/blog/pharos-releases-pacific-mainnet-v0-16-2'
date: "2026-10-29"
description: 'Pharos releases Pacific Mainnet v0.16.2 with execution-layer stability improvements, internal optimizations, and updated node-operator upgrade guidance.'
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Pharos%20Releases%20%20Pacific%20Mainnet%20v0.16.2.png
coverAlt: 'Pharos Releases Pacific Mainnet v0.16.2'
tags: ["Pharos", "Pacific Mainnet", "Node Upgrade", "Infrastructure"]
---


Pharos has released Pacific Mainnet client v0.16.2, a maintenance update focused on execution-layer stability, alongside internal optimizations that Pharos hasn't described in detail. The build ships as Docker image `pharos_community_v0.16.2_10e75201_1022` and doesn't require a hard fork. Operators mainly need to clarify one step: whether to refresh the local `VERSION` file during the upgrade, because the available instructions disagree.

## What changed

Pharos lists two changes for v0.16.2: improved execution-layer stability and additional internal optimizations. There's no granular changelog. The notice doesn't identify a specific execution bug, and Pharos hasn't published enough detail to claim measurable gains in throughput or latency, or to point to any change in consensus behavior.

## Why this is not a hard fork

The current [`mainnet.version`](https://github.com/PharosNetwork/resources/blob/main/mainnet.version) entry for `Pharos-Mainnet-260929` still shows version 16 and epoch 1746\. Only the referenced commit has moved, to `10e752016`, the same commit family as the `10e75201` in the v0.16.2 Docker tag. Nodes get a new client build with no step to SpecVersion 17, which is consistent with Pharos treating v0.16.2 as a maintenance upgrade rather than a protocol activation.

| Parameter | Current value |
| ----- | ----- |
| SpecVersion | `16` |
| Epoch | `1746` |
| Commit | `10e752016` |
| Client release | `v0.16.2` |

*SpecVersion, epoch and commit: Pharos `mainnet.version`. Client release: Pharos v0.16.2 release notice.*

## Upgrade notes for node operators

Pharos tells registered mainnet validators not to upgrade independently and to follow the scheduled upgrade plan instead. Nodes should be fully stopped before the Docker image or binary is replaced, then restarted and watched until block production is normal.

On Docker, the full image path is `public.ecr.aws/k2g7b7g1/pharos:pharos_community_v0.16.2_10e75201_1022`. Pharos's [Docker guide](https://docs.pharos.xyz/node-and-validator-guide/validator-node-deployment/using-docker-deployment) pins images by version and follows the same flow: replace the image tag, pull the new build, restart the service, and check node status. The release notice also calls for updating the `ops` tool and running `./ops health-check` after the upgrade to confirm the node version and SpecVersion. Pharos's [node-management documentation](https://docs.pharos.xyz/node-and-validator-guide/node-management) uses the same command for routine health checks.
