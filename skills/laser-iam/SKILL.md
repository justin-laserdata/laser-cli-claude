---
name: laser-iam
description: >-
  Organization identity and access under `org`. The organization's details and settings (delete protection, join policy, invitation rules), members, invitations, roles and their permissions, API keys with IP restrictions, and the cloud accounts used for VPC peering and BYOC.
---

# laser-iam

This skill covers who can do what in the organization:

- Organization details and settings: name, contact email, delete protection, join policy and invitation rules.
- Members: people who joined, with their roles.
- Invitations: pending invites by email.
- Roles: permission sets for members and API keys, granted at the organization, in every workspace, or in every environment.
- API keys: non-human credentials with a role or custom grants, optionally limited to one workspace and to source addresses.
- Cloud accounts: AWS accounts and GCP projects registered for VPC peering and BYOC.


## Organization

```sh
laser org show -o json
laser org summary -o json
laser org update --description "Payments platform" --email ops@example.com
laser org settings show -o json
laser org settings update --join-policy invite_only
laser org settings update --block-external-invitations true --enforce-domain-only-invitations true
```

`org update` takes `--name`, `--description`, `--email` and `--protected`. `settings update` changes `--protected`, `--join-policy` (`invite_only`, `open` or `request_to_join`), `--block-external-invitations` and `--enforce-domain-only-invitations`, and keeps whatever is left out. The invitation rules apply when the organization has a claimed email domain. Creating or deleting an organization needs the console.

## Members and invitations

```sh
laser org member list --active true -o json
laser org member permissions -o json
laser org member update 615380456123456790 --role 615380456123456801 --role 615380456123456802
laser org member update 615380456123456790 --active false
laser org member remove 615380456123456790 --yes
laser org invitation list -o json
laser org invitation create ana@example.com --role 615380456123456801
laser org invitation cancel 615380456123456811 --yes
```

`member update --role` replaces the member's roles with the repeated values. `--active` turns the member on or off. `member remove` takes away access to the organization and every workspace, environment and deployment in it. `member permissions` prints the catalog of grants a role can hold. Invitees accept in the console. Invitations use the current API key and require the organization member-management grant. Read `org role list` for role IDs before inviting. There is no separate resend command.

## Roles

```sh
laser org role list -o json
laser org role show 615380456123456801 -o json
laser org role create deploy-readers --workspace-permission environment:read --environment-permission deployment:read --environment-permission deployment:telemetry:read
laser org role update 615380456123456801 --name deploy-viewers
laser org role update 615380456123456801 --environment-permission deployment:read --environment-permission deployment:activity:read
laser org role assign 615380456123456801 615380456123456790 615380456123456791
laser org role revoke 615380456123456801 615380456123456791
laser org role members 615380456123456801 -o json
laser org role delete 615380456123456801 --yes
```

Each permission flag repeats. `--organization-permission` grants at the organization, `--workspace-permission` in every workspace, `--environment-permission` in every environment. `role update` replaces the grants of each scope you pass and keeps the others. A grant limited to one workspace or environment needs the API. Roles are `system`, `API key` or `custom`, and only custom roles change.

## API keys

```sh
laser org key list -o json
laser org key show 615380456123456821 -o json
laser org key whoami -o json
laser org key create ci-deployer --role 615380456123456801 --expires-in-days 90 --allowed-ip 203.0.113.7
laser org key create ingest -w main --workspace-scoped --role 615380456123456801
laser org key security 615380456123456821 --validate-ip true --allowed-ip 203.0.113.7 --allowed-ip 198.51.100.4
laser org key security 615380456123456821 --clear-allowed-ips
laser org key delete 615380456123456821 --yes
```

`key create` takes a `--role`, or custom grants with the same permission flags as `role create`. Keys expire after 30 days unless `--expires-in-days` says otherwise. `--workspace-scoped` limits the key to the selected workspace. `key security` replaces the allowed addresses, and `--clear-allowed-ips` removes them all. `key whoami` shows the key this CLI signs in with, its role and permissions. `keys` and `api-key` are aliases for `key`.

## Cloud accounts

```sh
laser org cloud-account list --cloud aws -o json
laser org cloud-account show 615380456123456831 -o json
laser org cloud-account create prod-aws --cloud aws --account-id 123456789012 --region eu-west-1 --iam-role-arn arn:aws:iam::123456789012:role/laserdata --external-id 7f3c2a
laser org cloud-account create analytics-gcp --cloud gcp --account-id example-project --vpc-network analytics --vpc-cidr 10.30.0.0/16
laser org cloud-account update 615380456123456831 --status inactive
laser org cloud-account delete 615380456123456831 --yes
```

`--account-id` takes the AWS account id or the GCP project id. AWS accounts take `--iam-role-arn`, `--external-id` and `--vpc-id`, GCP projects take `--vpc-network`, and both take `--vpc-cidr`. `update` changes only the flags you pass.

## Hard rules

1. `key create` prints the secret once. Don't run it yourself: print the command and have the user run it in their own terminal, so the secret never lands in this conversation. Tell them to store it in a secret manager at once.
2. Never print, repeat or log an API key secret.
3. `member update --role` and `role update` replace grants. Read the current state first (`member list`, `role show`), build the full set, show before and after, and wait for approval. To add one role to members without touching the others, use `role assign`.
4. Don't invent permission strings. Take them from `org member permissions`.
5. Removing a member, deleting a role, deleting a key, cancelling an invitation and deleting a cloud account are destructive: without a terminal the CLI refuses them unless `--yes` is passed. Show the target (for a key its name, role, workspace and expiry), ask the user to type its name or email, and only then add `--yes`.
6. Before deleting a role, list `role members` and check those members keep the access they need.
7. Prefer short-lived, IP-restricted keys for CI. Warn before creating a long-lived key with no allowed addresses.
8. Some actions need a signed-in browser session and refuse an API key: creating or deleting an organization, account settings, adding a card, and linking a marketplace purchase. Point the user at the console for those.

## Common asks

- "create a CI key for the deploy bot": find a fitting role (`role list`, `role show`), then hand the user the full command, such as `laser org key create deploy-bot --role 615380456123456801 --expires-in-days 90 --allowed-ip 203.0.113.7` with the CI egress address, to run themselves.
- "add a developer": `org invitation create <email> --role <developer role id>`.
- "someone left": confirm, `org member remove <id> --yes`, then check `org key list` for keys they created and rotate those.
- "rotate a leaked key": have the user create the replacement, then `org key delete <id> --yes` after approval. There is no in-place rotation.
- "what can my key do": `org key whoami -o json`.
- "register our AWS account for BYOC": `org cloud-account create` with the account id, role ARN and external id.

## Don't

- Don't paste key secrets or invitation links into the chat.
- Don't bulk-change members or roles. One confirmed change at a time.
- Don't confuse organization roles with deployment governance roles (see `/laser-cluster`).

Role commands accept an ID or an exact role name. A member carries role names, so use those names to open the role, then `org role members` to inspect assignments. The dashboard exposes a Members tab on each role page.
