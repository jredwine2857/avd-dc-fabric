# AVD DC Fabric - EVPN/VXLAN, CloudVision CI/CD

Greenfield Arista Validated Designs (AVD) source of truth for a 5-stage Clos
(super-spine + pods) EVPN/VXLAN fabric. Fully IaC: `git push` -> GitHub
Actions renders configs, then deploys them to CloudVision as a Service
(CVaaS) via a Workspace + Change Control.

## Routing design

| Layer    | Protocol | Notes |
|----------|----------|-------|
| Underlay | eBGP     | Unique ASN per device (or per pod-pair). Directly-connected point-to-point peering, no IGP. |
| Overlay  | eBGP EVPN | Unique ASN per leaf pair. Pod-spines and super-spines act as EVPN route-servers (`next-hop-unchanged`), relaying routes between leaf ASNs without terminating the overlay themselves. |

**Note on "iBGP underlay":** AVD's `underlay_routing_protocol` fabric
variable does not have a native `ibgp` option (only `ebgp`, `ospf[-ldp]`,
`isis` variants, or `none` - iBGP is only valid as an **overlay** protocol).
This repo uses AVD's default, most battle-tested pattern instead: eBGP for
both underlay and overlay, which is what most Arista hyperscale reference
designs run in production. Switching to an iBGP overlay with spine
route-reflectors is a one-line change (`overlay_routing_protocol: ibgp` in
`group_vars/FABRIC/fabric_variables.yml`) plus a shared overlay `bgp_as` per group.

## Topology

```
                [ dc1-ss1 ]        [ dc1-ss2 ]      <- SUPERSPINES (AS 65000)
                 /   |   \          /   |   \
     ___________/    |    \________/    |    \___________
    /                 |                 |                \
[pod1-spine1]   [pod1-spine2]   [pod2-spine1]      [pod2-spine2]
   AS 65001         AS 65001        AS 65002           AS 65002
        \               /                \                /
    [pod1-leaf1a/b]  [pod1-leaf2a/b]   [pod2-leaf1a/b]  [pod2-leaf2a/b]
     AS 65101-65150 (per leaf pair)     AS 65151-65200 (per leaf pair)
```

Only 2 pods x 2 leaf pairs are scaffolded as a working example. Scale out by
copying a `group_vars/POD<N>/` file and adding the group to `inventory.yml`
(see the comments in `group_vars/POD1/pod1.yml`).

## ASN & id plan

Every node has a globally-unique `id` (not just unique within its group),
which is what makes it safe to reuse shared IP pools (loopbacks, MLAG peer
links) across every group instead of carving out a separate pool per pod.

| Role             | id range  | bgp_as        |
|------------------|-----------|---------------|
| Super-spines      | 101-102   | 65000 (shared) |
| Pod1 spines       | 111-112   | 65001 (shared) |
| Pod1 leaf pairs   | 121-124   | 65101-65150 (auto, per leaf pair) |
| Pod2 spines       | 211-212   | 65002 (shared) |
| Pod2 leaf pairs   | 221-224   | 65151-65200 (auto, per leaf pair) |

Adding Pod3: use ids 3xx and `bgp_as: 65003` (spine) / `65201-65250` (leafs).

## Repo layout

```
inventory.yml                        # SUPERSPINES + POD1/POD2 groups
group_vars/
  FABRIC/                            # fabric-wide design vars (routing protocols,
                                      # default node types/interfaces, mgmt, AAA, NTP)
  SUPERSPINES/                       # super-spine tier
  POD1/, POD2/                       # per-pod spine + leaf definitions
  EVPN_SERVICES/                     # tenants/VRFs/VLANs/VNIs - empty placeholder,
                                      # add your services here when ready
playbooks/
  build.yml                          # eos_designs + eos_cli_config_gen (local render only)
  deploy.yml                         # cv_deploy -> pushes to CVaaS
  validate.yml                       # anta_runner -> post-deploy live validation via eAPI
intended/                            # generated structured configs + device configs (gitignored)
documentation/                       # generated fabric docs (gitignored)
.github/workflows/
  ci.yml                             # PR: renders configs, uploads as build artifact for review
  deploy.yml                         # main: renders + deploys to CVaaS (gated by a GitHub Environment)
```

## Before you run this for real

This scaffold builds and validates cleanly, but several placeholders exist
because hardware/site specifics weren't provided - replace these first:

- **`platform: CHANGE_ME_PLATFORM_SKU`** in every `group_vars/*/[a-z]*.yml`
  file - set to your actual Arista EOS platform (e.g. `7050X3`, `7280R3`).
- **`default_interfaces` Ethernet ranges** in `group_vars/FABRIC/fabric_variables.yml`
  and the per-node `uplink_switch_interfaces` in `group_vars/POD*/` - these
  assume generic port numbering and need to match your real cabling/platform.
- **Management IPs** (`ansible_host` in `inventory.yml`, `mgmt_ip` in
  group_vars, `mgmt_gateway` in FABRIC vars) - currently `192.0.2.0/24`
  (TEST-NET-1, intentionally non-routable placeholders).
- **IP pools** (loopback, VTEP loopback, uplink p2p, MLAG) - currently in
  the `10.255.0.0/16` placeholder range; resize/re-plan for your real scale.
- **Tenants/VRFs/VLANs/VNIs** in `group_vars/EVPN_SERVICES/evpn_services.yml`
  - intentionally left empty.

## Required GitHub configuration

**Secrets** (Settings -> Secrets and variables -> Actions):

| Secret | Used by | Purpose |
|--------|---------|---------|
| `CV_SERVER` | deploy.yml | CVaaS cluster FQDN for your region, e.g. `www.cv-prod-us-central1-a.arista.io` |
| `CV_TOKEN` | deploy.yml | CVaaS service-account API token ([create one in CloudVision](https://www.arista.io/help/articles/provisioning-studios-built-in-inventory)) |
| `EOS_USERNAME` / `EOS_PASSWORD` | validate.yml (optional, if run in CI) | eAPI creds for live ANTA validation |

**Environment**: create a `production` environment (Settings ->
Environments) and add required reviewers, so `deploy.yml` pauses for
approval before pushing to CloudVision.

## Running locally

```bash
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml

# Render structured configs, EOS configs, and docs into intended/ and documentation/
ansible-playbook playbooks/build.yml

# Push to CVaaS (stages a Workspace; leaves the Change Control pending approval
# unless you pass -e cv_run_change_control=true)
CV_SERVER=www.cv-prod-us-central1-a.arista.io CV_TOKEN=*** \
  ansible-playbook playbooks/deploy.yml

# Validate live fabric state via eAPI (ANTA)
EOS_USERNAME=*** EOS_PASSWORD=*** ansible-playbook playbooks/validate.yml
```

## CI/CD flow

1. Open a PR changing `group_vars/`/`inventory.yml` -> `ci.yml` renders
   configs and docs, uploads them as a build artifact for reviewers to diff.
2. Merge to `main` -> `deploy.yml` re-renders and pushes to CVaaS as a
   Workspace (and, if `production` environment reviewers approve /
   `run_change_control` is set, runs the Change Control). Otherwise the
   Change Control is left pending for manual approval in CloudVision.
