---
name: laser-access
description: >-
  Control which networks reach a deployment with `deployment connect`. Access rules (CIDR blocks on chosen protocols, with expiry), the platform network, VPC peering, and private endpoints through AWS PrivateLink or GCP Private Service Connect.
---

# laser-access

Access rules are the deployment's firewall: IPv4 CIDR blocks admitted on chosen protocols. A deployment admits no traffic from an address until a rule covers it. The data access proxy needs no rule from you: LaserData adds the rule it needs while data access is on. Direct connections, SDK clients and `--direct` CLI commands need a rule for the caller's address.

A rule admits an address. It hands out no login. A client that connects directly signs in with the deployment admin login `ld-admin` from `connect show`, which reads and writes data but cannot manage users or the server. Root needs `deployment:manage` as well. Both logins, and their rotation, are in `/laser-credentials`.

## Access rules

```sh
laser deployment connect access-rule list -d orders -o json
laser deployment connect access-rule add -d orders --name office --cidr 203.0.113.7/32 --protocol tcp --protocol http --remarks "Office VPN"
laser deployment connect access-rule add -d orders --name contractor --cidr 198.51.100.0/24 --protocol tcp --valid-to 2026-12-31T23:59:59Z --remarks "Audit access until year end"
laser deployment connect access-rule delete -d orders 615380456123456881 --yes
laser deployment connect access-rule allow-my-ip -d orders --protocol tcp
```

- `--name` is required and unique within the deployment, ignoring case.
- `--cidr` is an IPv4 CIDR block and repeats for several.
- `--protocol` is `http`, `tcp`, `udp` or `websocket` and repeats. Every protocol is opened when it is left out, so pass the protocols the client uses.
- `--valid-to` sets an expiry in RFC 3339, in the future. `--remarks` records why the rule exists.

The plan limits the number of rules per deployment, and rules LaserData creates for VPC peering count toward it. See <https://docs.laserdata.com/networking/access-rules>.

`access-rule allow-my-ip` admits the address LaserData sees the CLI call from, as a `/32` rule for 24 hours that counts toward the plan limit. A second call replaces it. `--protocol` repeats (`tcp`, `http`, `websocket`, `udp`). Use `tcp` before a Laser SDK app connects from the same machine. It answers that the address is already allowed when another rule covers it. A Free deployment carries the global rule `0.0.0.0/0`, so its clients need no rule.

## Network, peering and private endpoints

```sh
laser deployment connect network -d orders -o json
laser deployment connect peering list -d orders -o json
laser deployment connect peering create -d orders --name prod-vpc --peer-vpc-cidr 10.20.0.0/16 --peer-vpc-id vpc-0abc --peer-owner-id 123456789012 --peer-region eu-west-1
laser deployment connect peering create -d orders --name analytics-vpc --peer-vpc-cidr 10.30.0.0/16 --peer-vpc-name analytics --peer-project-id example-project
laser deployment connect peering instructions -d orders 615380456123456911
laser deployment connect peering delete -d orders 615380456123456911 --yes
laser deployment connect private-link list -d orders -o json
laser deployment connect private-link create -d orders --name orders-endpoint --allowed-principal arn:aws:iam::123456789012:root
laser deployment connect private-link create -d orders --name orders-psc --connection-preference accept_manual --consumer-accept-list example-project
laser deployment connect private-link instructions -d orders 615380456123456921
laser deployment connect private-link connections -d orders 615380456123456921
laser deployment connect private-link delete -d orders 615380456123456921 --yes
```

- `network` shows the platform network id and CIDR block. A peer network must not overlap it.
- VPC peering needs `--name` and `--peer-vpc-cidr`. AWS takes `--peer-vpc-id`, `--peer-owner-id` and `--peer-region` (the deployment's region by default). GCP takes `--peer-vpc-name` and `--peer-project-id`. `instructions` prints the steps that complete the connection on the user's side.
- Private endpoints are AWS PrivateLink endpoint services or GCP Private Service Connect attachments. AWS takes `--acceptance-required` and repeated `--allowed-principal`. GCP takes `--connection-preference` (`accept_manual` or `accept_automatic`), repeated `--consumer-accept-list` and `--enable-proxy-protocol`. Only GCP has `instructions` and `connections`.
- Peering starts at the Performance managed tier and private endpoints need Enterprise.

## Hard rules

1. Widening access has a large blast radius. Before adding `0.0.0.0/0` or any range wider than the user asked for, warn and ask for explicit confirmation. Recommend the narrowest range that works.
2. Every rule gets a `--name` and `--remarks`. If the user gave neither, ask.
3. Never guess an office or VPC range. Ask for it.
4. Rule, peering and private endpoint deletes cut traffic at once. Show the name and range, ask the user to confirm, and only then run the delete with `--yes`. Without a terminal the CLI refuses a delete that lacks `--yes`.
5. Listing rules needs `deployment:access:read`, adding and deleting need `deployment:access:manage`. Listing peering and private endpoints needs `deployment:network:read`, creating and deleting them needs `deployment:network:manage`.
6. Never print a password or connection string while you set up a client's access. The client's login comes from `/laser-credentials`, masked in the chat, and the user reveals it in their own terminal.

## Common asks

- "let me connect from my IP": `access-rule allow-my-ip --protocol tcp` with the protocols the client uses. It admits the address LaserData sees for 24 hours. For a lasting rule, confirm the address with the user and use `access-rule add --name <user>-laptop --cidr <ip>/32 --protocol tcp --remarks "<user> laptop"`. The client then signs in with the deployment admin login from `connect show`.
- "my client stopped connecting": check whether its rule expired (`access-rule list -o json`, `valid_to`, and `allow-my-ip` rules last 24 hours), then whether the login was rotated (`laser deployment monitor activity -d orders`). After an admin rotation every client signed in as `ld-admin` is cut off, live sessions included, and needs the new password from `/laser-credentials`. After a root rotation, root's live sessions keep working until they end or expire.
- "what can reach orders right now": `access-rule list -o json`, render name, CIDR blocks, protocols and expiry. Flag rules whose `valid_to` has passed.
- "lock this deployment down": list the rules, confirm each one, delete one at a time.
- "peer with our AWS VPC": `connect network` for the platform CIDR, check the user's range does not overlap, `peering create`, then `peering instructions` for the user's side.

## Don't

- Don't suggest `0.0.0.0/0` casually.
- Don't open every protocol when the client uses one. Pass `--protocol`.
- Don't bulk-delete rules without per-entry confirmation.
