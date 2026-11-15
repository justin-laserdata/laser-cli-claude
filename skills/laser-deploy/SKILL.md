---
name: laser-deploy
description: >-
  Create and run deployments. Translates a request into a starter, managed or BYOC `deployment create` command with every value discovered from `deployment create options`, prices it first with `create preview`, follows it with `deployment watch`, and handles the lifecycle under `deployment settings` (rename, move, protection, data access, metadata sync, retention, spend limit, plan upgrade) with requested against applied and `--wait`, then `deployment delete`.
---

# laser-deploy

Goal: turn a request like "spin up a deployment in eu-west-1 on the Standard tier" into a concrete `deployment create` command with every flag filled from discovery and confirmed by the user, then carry the deployment through its life.

## Hard rules

1. Never invent a cloud, region, compute profile, cluster kind, storage type or environment. Discover them at runtime. Only what `deployment create options` returns for this organization counts.
2. Running from Claude Code there is no terminal, so the CLI neither prompts for missing values nor shows its summary: a create runs the moment it is sent. Pass every value as a flag, print the full command, and wait for the user's approval.
3. Price a paid deployment with `deployment create preview` before creating it. Say it is an estimate from list prices, not a quote.
4. Parse `-o json`, never the table.
5. If `laser auth status` fails, stop and point the user at `/laser-onboard`.
6. Never pass an API key or a password on a command line.
7. Destructive changes (delete, move, data access off, metadata sync off) and spend-limit changes need the user to type the deployment name in the chat first. Without a terminal the CLI refuses the destructive ones unless `--yes` is passed, so add `--yes` only after that approval. Plan upgrades change the bill: get an explicit yes for those too.

## Discover

```sh
laser auth status
laser org show -o json
laser env list -o json
laser deployment create options clouds -o json
laser deployment create options regions --cloud aws -o json
laser deployment create options tiers --cloud aws --region eu-west-1 -o json
laser deployment create options clusters --cloud aws --region eu-west-1 -o json
laser deployment create options storages --cloud aws --region eu-west-1 -o json
```

Each option row has a `key`, a `name` and, for tiers, clusters and storages, an `available` flag for this organization's plan. Pass the `key` of an available row as the flag value: `--tier` takes a compute profile key, `--cluster` a cluster key, `--storage-type` a storage key. A tier row lists the cluster kinds and storages it supports. `org show` returns the plan.

Managed tiers are `standard`, `performance` and `enterprise`. `multi_az` availability needs Enterprise, VPC peering starts at Performance, and private endpoints need Enterprise.

## Price

```sh
laser deployment create preview --cloud aws --region eu-west-1 --managed-tier standard --tier small --storage-type network_balanced --storage-size-gb 100 -o json
laser pricing get --rates -o json
laser pricing estimate managed --managed-tier standard --cloud aws --compute-profile-id small --storage-profile-id network_drive --storage-gb-per-node 100 --throughput-mb-per-second 5
laser pricing estimate byoc --vcpus-per-node 8
```

`create preview` estimates the monthly price of a managed deployment without creating anything. `--managed-tier` is required unless the compute profile is free. `pricing` needs no sign-in and prices from published rates. `--throughput-mb-per-second` is estimated symmetric traffic, not a reservation.

## Create

### Starter (free, the simplest path)

```sh
laser deployment create starter --cloud aws --region us-east-1 --new-env sandbox -o json
```

Ask AWS or GCP and propose a region, `us-east-1` by default on AWS. `--env` deploys into an existing environment, `--new-env` creates one. `--name` is optional, LaserData picks one.

### Managed

```sh
laser deployment create managed --env production --name orders --cloud aws --region eu-west-1 --managed-tier standard --tier small --cluster cluster --storage-type network_balanced --storage-size-gb 100 --availability-mode single_az --spend-limit 500 -o json
```

Required in practice: `--env`, `--name`, `--cloud`, `--region`, `--managed-tier`, `--tier`, `--storage-type`, `--storage-size-gb` (per node). Paid deployments are a three node cluster. Optional: `--availability-mode` (`single_az` or `multi_az`), `--network-scope`, `--target-network-tput` in KB/s, `--dedicated`, `--protected`, `--encrypted`, `--public-ip-enabled`, `--subdomain-enabled`, `--telemetry-days`, `--spend-limit` in USD per month, and `--data-access` or `--no-data-access`. A managed deployment needs a public IP for data access.

### BYOC (bring your own cloud)

Three steps, never skip the second:

```sh
laser deployment create byoc-setup --env production --cloud aws --region eu-west-1 -o json
laser deployment create byoc-validate --env production --supervisor-url https://byoc-check.example.com --cloud aws --region eu-west-1 --aws-account-id 123456789012 --aws-identity-arn arn:aws:iam::123456789012:role/laserdata --aws-external-id 7f3c2a --aws-vpc-id vpc-0abc -o json
laser deployment create byoc --env production --name orders-byoc --cloud aws --region eu-west-1 --tier large --cluster cluster --storage-type network_balanced --storage-size-gb 500 --availability-mode single_az --aws-account-id 123456789012 --aws-identity-arn arn:aws:iam::123456789012:role/laserdata --aws-external-id 7f3c2a --aws-vpc-id vpc-0abc --aws-vpc-cidr 10.0.0.0/16 -o json
```

1. `byoc-setup` prints the cloud setup the deployment needs, the external id, and the URL that `byoc-validate` takes as `--supervisor-url`. The user applies the setup in their own cloud console.
2. `byoc-validate` checks the credentials before anything is provisioned. A failure here is cheap, a failure at create time leaves a half-built deployment.
3. `byoc` creates it. Same cloud flags as validate plus `--aws-vpc-cidr`.

GCP takes `--gcp-project-id`, `--gcp-service-account-email` and `--gcp-vpc-name` in place of the AWS flags. The cloud account can also be registered once with `org cloud-account create` (see `/laser-iam`).

### After create

```sh
laser deployment watch -d orders -o json
laser deployment watch -d orders --timeout-mins 30 -o json
```

`watch` waits until the running operation ends and every requested settings change is applied. It reads every 5 seconds (`--interval-secs` 2 to 60) and gives up after 60 minutes (`--timeout-mins`) with exit code 4. It exits 0 when the operation ends either way, so read `status` from the JSON: `initialized` is ready, `failed` is a failure to hand to `/laser-troubleshoot`. The `changes` array lists each setting with `requested`, `applied` and `state`.

Don't loop on `deployment list` waiting for ready. `watch` is the primitive.

## Operate

```sh
laser deployment list -o json
laser deployment list --health degraded,unavailable --sort storage_used_percent -o json
laser deployment show -d orders -o json
laser deployment use orders
laser deployment seed -d orders --wait
```

`list` shows every deployment of the workspace with live health (`healthy`, `degraded`, `unavailable`, `unknown`), storage, connections and agent sessions, merged across zones. A zone that does not answer makes the list incomplete, not empty. `show` is the Overview tab: details, settings requested against applied, member health and nodes. `seed` adds demo data, a demo stream with projections and agent sessions.

## Settings

```sh
laser deployment settings show -d orders -o json
laser deployment settings rename orders-eu -d orders --wait -o json
laser deployment settings move -d orders --to-env production --wait --yes
laser deployment settings update -d orders --description "Order events"
laser deployment settings update -d orders --protected true
laser deployment settings data-access on -d orders --wait
laser deployment settings data-access off -d orders --wait --yes
laser deployment settings metadata-sync off -d orders --yes
laser deployment settings metadata-sync off -d orders --purge --yes
laser deployment settings retention -d orders --telemetry-days 30
laser deployment settings spend-limit -d orders --limit 5000
laser deployment settings spend-limit -d orders --clear
laser deployment settings plan show -d orders -o json
laser deployment settings plan upgrade -d orders --managed-tier performance --wait
laser deployment settings plan upgrade -d orders --tier large --wait
laser deployment settings plan upgrade -d orders --storage-type network_balanced --storage-size-gb 1000 --wait
```

Renames, moves, data access, metadata sync and plan upgrades are accepted at once and applied by the deployment's zone shortly after. These commands print the requested value against the applied one. In JSON that is `requested`, `applied` and `phase` (`applied`, `pending`, `failed` or `unknown`), with a `note` when there is one. `--wait` reads every 5 seconds until the change is applied or refused, and a refused change exits with code 4. `--wait` has no time limit of its own: for a slow change, run without it and follow with `deployment watch --timeout-mins`.

- `rename` keeps the address and server names.
- `move` moves the deployment to another environment of the workspace. Permissions then follow the new environment.
- `update` changes `--description` or `--protected`. Turning protection off is an organization owner action.
- `data-access off` routes data views straight to the deployment, and every caller then needs an access rule.
- `metadata-sync off` stops syncing streams, topics and catalog metadata. `--purge` also deletes the records LaserData stores, which cannot be undone.
- `plan upgrade` changes `--managed-tier`, `--tier`, or `--storage-type` with `--storage-size-gb`. Storage cannot shrink. Preview the new price first.

## Delete

```sh
laser deployment delete -d payments --yes
laser deployment delete -d payments --code 482913
```

Show the deployment (`laser deployment show -d payments -o json`), say that its data goes with it, and ask the user to type its name. Then run the delete with `--yes`. A protected deployment emails a confirmation code to the organization email and the command exits asking for it. Ask the user for the code and run the delete again with `--code`.

## Example

User: "deploy orders in production, aws eu-west-1, standard tier, small, 100 GB network storage".

1. `laser env list -o json` to confirm `production` exists.
2. `laser deployment create options tiers --cloud aws --region eu-west-1 -o json` and `laser deployment create options storages --cloud aws --region eu-west-1 -o json` to confirm `small` and the storage key are available.
3. `deployment create preview` with the same values, then show the monthly estimate.
4. Print the full `deployment create managed` line and ask `run this? (y/N)`. Wait.
5. On yes, run it, then `laser deployment watch -d orders -o json` and report `status`.

## Don't

- Don't add `--silent` to a create or a settings change. The user wants the result.
- Don't promise a tier, storage or availability mode without the options check.
- Don't create a deployment the user did not ask for, not even a starter "to test".
- Don't delete and recreate a deployment to fix it. Hand off to `/laser-troubleshoot` first.

For a free deployment, read regions with `deployment create options regions --cloud CLOUD --tier free`. Use only the returned regions. Never use the general region list or invent a regional fallback. The API filters enabled regions and free-tier availability, which can leave one region per cloud.
