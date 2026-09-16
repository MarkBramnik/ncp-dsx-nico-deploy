# NICo User Guide \- operational perspective

This document summarizes the main flows which the nico user/administrator can execute.   
Each flow contains a short explanation and a series of rest endpoints/nico-cli actions required to be invoked to perform this flow. Disclaimer: only some rest endpoints were verified, we’ll probably change them as we’ll start to actually work with NICo and have real data centers with real servers having real DPUs (also the NICO’s versions will evolve, so these APIs will change naturally). At the end of the document you’ll also find a Notion Vocabulary \- a list of notions with short definitions and relations between different notions.

# NVidia Repo of NICo:

[https://github.com/dsx-ai-factory/infra-controller](https://github.com/dsx-ai-factory/infra-controller)

# 

# Main Flows

NICo (NVIDIA Infra Controller) is a bare-metal infrastructure management platform. It exposes a REST API (`/v2/org/{org}/nico/...`) and a companion CLI (`nicocli`). Every flow below can be exercised with either `curl` or `nicocli`; both are shown.

This guide follows the natural lifecycle: **authenticate → bootstrap → register a site → onboard hardware → configure compute → provision instances**.

---

## Environment setup

Set these shell variables once. Every example below references them.

```shell
export API_URL="https://nico-rest-api-nico-rest.<cluster-domain>"
export KC_URL="https://keycloak-rhbk-operator.<cluster-domain>"
export ORG="ncx"          # your org name — derived from the Keycloak realm role prefix
```

To find your cluster domain:

```shell
oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}'
```

---

## 

## Flow 1 — Authentication

### What it does
Obtains a secure JSON Web Token (JWT) using the OAuth2 Machine-to-Machine (M2M) flow via Keycloak, and configures the local nicocli context with those credentials.

### When to invoke
Executed at the very beginning of any session or automation script, and periodically when token rotation or expiration occurs.

### What you want to achieve

Get an API bearer token so every subsequent call can be authorized. NICo uses Keycloak OIDC. There are two grant types:

- **M2M (service account)** — `client_credentials`, used by automation, CI, and the `ncx-service` Keycloak client. Long-lived enough for scripted workflows.
- **Human user** — `password` or browser PKCE, used by engineers logging in with personal credentials.

### M2M — service account token (scripted)

Fetch the `ncx-service` client secret from the Keycloak admin API, then exchange it for a token. The secret can contain `+`/`/` characters that `curl -d` corrupts; always use `--data-urlencode`.

```shell
# 1. Get an admin token for the Keycloak master realm
_ADMIN_USER=$(oc get secret keycloak-admin-secret -n rhbk-operator \
  -o jsonpath='{.data.username}' | base64 -d)
_ADMIN_PASS=$(oc get secret keycloak-admin-secret -n rhbk-operator \
  -o jsonpath='{.data.password}' | base64 -d)

_ADMIN_TOKEN=$(curl -sk -X POST "$KC_URL/realms/master/protocol/openid-connect/token" \
  -d grant_type=password -d client_id=admin-cli \
  -d "username=$_ADMIN_USER" -d "password=$_ADMIN_PASS" \
  | jq -r .access_token)

# 2. Look up the ncx-service client secret
_CLIENT_UUID=$(curl -sk -H "Authorization: Bearer $_ADMIN_TOKEN" \
  "$KC_URL/admin/realms/nico/clients?clientId=ncx-service" | jq -r '.[0].id')

_CLIENT_SECRET=$(curl -sk -H "Authorization: Bearer $_ADMIN_TOKEN" \
  "$KC_URL/admin/realms/nico/clients/$_CLIENT_UUID" | jq -r .secret)

# 3. Exchange client credentials for an API token
export TOKEN=$(curl -sk -X POST "$KC_URL/realms/nico/protocol/openid-connect/token" \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "client_id=ncx-service" \
  --data-urlencode "client_secret=$_CLIENT_SECRET" \
  | jq -r .access_token)

echo "Token length: ${#TOKEN}"
```

### nicocli — interactive login

```shell
# Write a default config file (one-time)
nicocli init

# Login interactively (opens browser or prompts for credentials)
nicocli login \
  --base-url "$API_URL" \
  --keycloak-url "$KC_URL" \
  --keycloak-realm nico \
  --client-id ncx-service \
  --org "$ORG"
```

After login, `nicocli` stores the token in `~/.nico/config.yaml` and reuses it automatically. All subsequent `nicocli` examples assume this config is in place.

### Verify the token is working

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/user/current" | jq .
```

Example response (confirmed on cluster):

```json
{
  "id": "91ecb18e-889c-421f-9e57-5ac9fcff9599",
  "firstName": "ncx-service",
  "lastName": "",
  "email": "",
  "created": "2026-09-02T08:33:27.117095Z",
  "updated": "2026-09-14T15:31:26.676346Z"
}
```

---

## 

## Flow 2 — Bootstrap the Organization

### What it does
Initializes the multi-tenancy layer by registering the primary Infrastructure Provider (the owner of the physical hardware) and setting up the initial Tenants (logical consumers, e.g., AI Dev Team), followed by a health check on the core controller nodes.

### When to invoke
Run exactly once during **Day-0 initialization** of a brand-new NICo cluster instance.

### What you want to achieve

The very first time NICo is deployed, no Infrastructure Provider or Tenant entity exists. These GET endpoints auto-create both on first call. Nothing else in the API works until these are initialized.

### Infrastructure Provider (Provider Admin role)

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/infrastructure-provider/current" | jq .
```

Example response:

```json
{
  "id": "7510b0d0-7499-4d67-af75-aa23978c22ed",
  "org": "ncx",
  "orgDisplayName": "ncx",
  "created": "2026-09-02T08:33:27.121108Z",
  "updated": "2026-09-02T08:33:27.121109Z"
}
```

### Tenant (Tenant Admin role)

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/tenant/current" | jq .
```

Example response:

```json
{
  "id": "800dc0da-85c3-4f3f-a79d-b89032cf948f",
  "org": "ncx",
  "orgDisplayName": "ncx",
  "created": "2026-09-02T08:33:27.670646Z",
  "updated": "2026-09-14T15:31:27.237854Z",
  "capabilities": {
    "targetedInstanceCreation": true
  }
}
```

### Platform-wide stats

```shell
# Infrastructure Provider view — machines, IP blocks, tenant accounts
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/infrastructure-provider/current/stats" | jq .

# Tenant view — instances, VPCs, subnets
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/tenant/current/stats" | jq .
```

Save the IDs — they appear in every subsequent request:

```shell
export INFRA_PROVIDER_ID=$(curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/infrastructure-provider/current" | jq -r .id)

export TENANT_ID=$(curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/tenant/current" | jq -r .id)
```

---

## Flow 3 — Site Registration

### What it does
Registers a physical location (Site-ID), establishes a secure trust pairing relationship between the core controller and the local site agent, and tracks its lifecycle until it reaches an active/paired state.

### When to invoke
When a new data center room, edge cluster, or physical rack group is physically connected to the infrastructure network.

### What you want to achieve

A **Site** represents a physical location where bare-metal hardware lives and where the NICo site agent runs. The Provider creates sites; Tenants cannot. After creation the site is `Pending` until the site agent connects and pairs — at which point it becomes `Registered` and `isOnline: true`.

### Create a site

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/site" \
  -d '{
    "name": "dc-amsterdam-01",
    "description": "Primary Amsterdam data center",
    "location": "Amsterdam, NL",
    "contact": "ops@example.com"
  }' | jq .
```

Key response fields:

- `id` — save this as `$SITE_ID`; used in almost every subsequent call
- `registrationToken` — passed to the site agent during its deployment
- `status` — starts as `Pending`, becomes `Registered` after agent pairing

```shell
# nicocli equivalent
nicocli site create \
  --name dc-amsterdam-01 \
  --description "Primary Amsterdam data center"
```

### List all sites

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/site" | jq .
```

```shell
nicocli site list
```

### Inspect a specific site

```shell
export SITE_ID="<id from create response>"

curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/site/$SITE_ID" | jq '{
    name, status, isOnline, siteControllerVersion, siteAgentVersion, statusHistory
  }'
```

Example response (confirmed on cluster, site `my-site`):

```json
{
  "name": "my-site",
  "status": "Registered",
  "isOnline": true,
  "siteControllerVersion": null,
  "siteAgentVersion": null,
  "statusHistory": [
    { "status": "Registered", "message": "Site has been successfully paired" },
    { "status": "Pending",    "message": "received site creation request, pending pairing" }
  ]
}
```

```shell
nicocli site get $SITE_ID
```

---

## Flow 4 — Pre-register Expected Machines

### What it does
Populates the NICo inventory database with MAC addresses, serial numbers, and target rack positions for upcoming physical assets before they communicate with the network.

### When to invoke
When a new shipment of servers arrives at the dock or when planning data center expansions via bill-of-materials (BOM) files.

### What you want to achieve

Before physical hardware is racked and powered on, you tell NICo what to expect. An **Expected Machine** is a pre-registration record keyed on the BMC MAC address (and optionally the chassis serial). When NICo's DHCP/PXE stack discovers the hardware and the MAC matches, the machine is automatically onboarded and linked to this record.

| *Sidenote:* why do we need such a flow if we have discovery: |
| :---- |
| **In short:** This flow is required to supply to the system what is “expected” to be in the datacenter, while the discovery flow shows what “really” is in there. Here are the main reasons for such a separation. This flow: **Acts as the physical master blueprint** by defining exactly what hardware *should* be in the data center before it arrives. **Prevents costly human placement errors** by cross-referencing live discovered serial numbers against planned rack locations. **Enforces zero-trust hardware security** by quarantining unauthorized or rogue devices that plug into switch ports. **Pre-allocates network resources** like static IPs, DNS entries, and switch configurations before servers are unboxed. **Tracks the physical deployment pipeline** to identify which purchased nodes are racked versus those still in transit. **Automates validation metrics** by matching auto-discovered component layouts against the vendor's shipping manifest. **Discovery finds what *is* there**, while **Expected Machines defines what *must* be there** to ensure structural integrity.  |

### Pre-register a machine

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/expected-machine" \
  -d '{
    "siteId": "'"$SITE_ID"'",
    "name": "node-01",
    "bmcMacAddress": "aa:bb:cc:dd:ee:01",
    "defaultBmcUsername": "root",
    "defaultBmcPassword": "factory-password",
    "chassisSerialNumber": "SN1234567",
    "skuId": "'"$SKU_ID"'"
  }' | jq .
```

Key fields:

- `bmcMacAddress` — the MAC address NICo uses to match the physical machine when it first contacts the DHCP server
- `defaultBmcPassword` — the factory-default BMC credential NICo uses to connect before rotating credentials
- `skuId` — optional; links this machine to a hardware SKU definition

```shell
nicocli expected-machine create \
  --site-id "$SITE_ID" \
  --name node-01 \
  --bmc-mac-address aa:bb:cc:dd:ee:01 \
  --default-bmc-username root \
  --default-bmc-password factory-password
```

### List expected machines for a site

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/expected-machine?siteId=$SITE_ID" | jq .
```

```shell
nicocli expected-machine list --site-id "$SITE_ID"
```

---

## Flow 5 — Discover & Inspect Machines

### What it does
Triggers an active/passive scan via BMC/Redfish to gather a real-time hardware snapshot (CPU cores, RAM size, DPU versions, NVLink connectivity, and GPU topologies) and monitors its lifecycle transitions.

### When to invoke
Right after new bare-metal servers are mounted into the rack, wired up, and powered on for the first time.

### What you want to achieve

Once hardware is racked and powered on, NICo discovers it via DHCP/PXE. Discovered machines appear in the machine list. Use this flow to see what has been onboarded, check machine health, and understand the machine's network interfaces and GPU inventory.

### List all machines

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/machine" | jq .

# Filter by site
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/machine?siteId=$SITE_ID" | jq .
```

```shell
nicocli machine list
nicocli machine list --site-id "$SITE_ID"
```

### Inspect a machine

```shell
export MACHINE_ID="<id from machine list>"

curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/machine/$MACHINE_ID" | jq .

# Network interfaces
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/machine/$MACHINE_ID/interface" | jq .

# InfiniBand interfaces
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/machine/$MACHINE_ID/infiniband-interface" | jq .

# Machine status history
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/machine/$MACHINE_ID/status-history" | jq .
```

### GPU statistics (across all machines in a site)

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/machine/gpu/stats?siteId=$SITE_ID" | jq .
```

```shell
nicocli machine gpu-stats --site-id "$SITE_ID"
```

---

## 

## Flow 6 — Power Control & BMC Reset

### What it does
Issues hardware-level power state commands (ON/OFF/Reboot) or cold-boots the BMC, requiring an explicit bypass flag if the system detects that an active workload (instance) is currently mapped to that machine.

### When to invoke
During hardware troubleshooting, firmware crashes, or prior to re-provisioning a server that requires a physical power cycle.

### What you want to achieve

Control machine power state and reset the BMC (Baseboard Management Controller) remotely. If a machine has an Instance attached you must explicitly acknowledge it to prevent accidental disruption — this is the `acknowledgeAttachedInstance` safety flag.

### Power on / off / reset

```shell
# Power on
curl -sk -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/machine/$MACHINE_ID/power" \
  -d '{"action": "on"}' | jq .

# Power off
curl -sk -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/machine/$MACHINE_ID/power" \
  -d '{"action": "off"}' | jq .

# Reset (reboot)
curl -sk -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/machine/$MACHINE_ID/power" \
  -d '{"action": "reset"}' | jq .

# If an Instance is attached, add acknowledgement
curl -sk -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/machine/$MACHINE_ID/power" \
  -d '{"action": "off", "acknowledgeAttachedInstance": true}' | jq .
```

```shell
nicocli machine power --machine-id "$MACHINE_ID" --action on
nicocli machine power --machine-id "$MACHINE_ID" --action off
nicocli machine power --machine-id "$MACHINE_ID" --action reset --acknowledge-attached-instance
```

### BMC reset

```shell
curl -sk -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/machine/$MACHINE_ID/bmc/reset" \
  -d '{"useIpmiTool": false, "acknowledgeAttachedInstance": false}' | jq .
```

```shell
nicocli machine bmc reset --machine-id "$MACHINE_ID"
```

---

## 

## Flow 7 — SKU Management

### What it does
Handles hardware blueprints (SKUs) by either converting a flawless discovered machine snapshot into an official reusable hardware profile or manually uploading a vendor specification layout.

### When to invoke
When establishing standard server archetypes (e.g., standardizing a 15-node cluster configuration for HGX H100s vs. A100 storage targets).

### What you want to achieve

A **SKU** (Stock Keeping Unit) describes a hardware configuration: device type, CPU, memory, GPU count, NIC specs, etc. SKUs are usually auto-derived when machines are discovered. You reference SKUs when pre-registering expected machines and when creating instance types.

### List discovered SKUs at a site

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/sku?siteId=$SITE_ID" | jq .
```

```shell
nicocli sku list --site-id "$SITE_ID"
```

### Create a SKU manually

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/sku" \
  -d '{
    "siteId": "'"$SITE_ID"'",
    "id": "dgx-h100-8gpu",
    "description": "DGX H100 — 8x H100 SXM5 80GB",
    "deviceType": "server",
    "components": {
      "gpu": [{"model": "H100", "count": 8, "memoryGb": 80}],
      "cpu": [{"model": "Intel Xeon Platinum 8480+", "count": 2, "cores": 60}],
      "memoryGb": 2048,
      "storageGb": 30720
    }
  }' | jq .
```

```shell
export SKU_ID="<id from SKU list or create>"
```

---

## 

## Flow 8 — Instance Types

### What it does
Map physical hardware SKUs into software-consumable "t-shirt sizes" or specialized tiers (e.g., gpu.8x.h100.large), binding qualified physical machines to these definitions.

### When to invoke
When defining the service catalog for end-users, ensuring that software requests map perfectly to correct physical hardware boundaries.

### What you want to achieve

An **Instance Type** is the compute capacity definition that a Tenant consumes when creating an Instance. It is created by the Provider, scoped to a Site, and describes what hardware capabilities a workload gets (GPU count, NIC type, etc.). Think of it as the "flavor" in cloud terminology.

### Create an instance type

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/instance/type" \
  -d '{
    "name": "h100-8gpu",
    "description": "8x H100 SXM5 bare metal",
    "siteId": "'"$SITE_ID"'",
    "labels": {"gpu": "h100", "count": "8"},
    "controllerMachineType": "server"
  }' | jq .

export INSTANCE_TYPE_ID="<id from response>"
```

```shell
nicocli instance-type create \
  --name h100-8gpu \
  --site-id "$SITE_ID" \
  --description "8x H100 SXM5 bare metal"
```

### List instance types

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/instance/type" | jq .
```

```shell
nicocli instance-type list
```

### Associate a machine with an instance type

Once a machine is discovered and an instance type exists, link them so the machine can be allocated to that type:

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/instance/type/$INSTANCE_TYPE_ID/machine" \
  -d '{"machineId": "'"$MACHINE_ID"'"}' | jq .
```

---

## 

## Flow 9 — Allocations

### What it does
Allocates specific physical node blocks or a quota of Instance Types exclusively to a designated Tenant group, isolating compute capacity.

### When to invoke
When a specific team or project requests a dedicated slice of the cluster for private training/inference workloads.

### What you want to achieve

An **Allocation** is how the Provider carves out capacity for a Tenant. The Provider assigns a set of machines (or an IP prefix) at a specific site to the Tenant. Until an allocation exists, the Tenant cannot create instances at that site.

### Create an allocation

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/allocation" \
  -d '{
    "name": "amsterdam-h100-pool",
    "description": "H100 pool for AI training workloads",
    "tenantId": "'"$TENANT_ID"'",
    "siteId": "'"$SITE_ID"'",
    "allocationConstraints": [
      {
        "instanceTypeId": "'"$INSTANCE_TYPE_ID"'",
        "count": 4
      }
    ]
  }' | jq .

export ALLOCATION_ID="<id from response>"
```

```shell
nicocli allocation create \
  --name amsterdam-h100-pool \
  --tenant-id "$TENANT_ID" \
  --site-id "$SITE_ID"
```

### List allocations

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/allocation" | jq .

nicocli allocation list
```

---

## 

## Flow 10 — SSH Keys & Key Groups

### What it does
Registers public SSH keys and aggregates them into key groups within the NICo database to allow seamless, key-based remote root access during bare-metal deployment.

### When to invoke
When onboarding new infrastructure engineers or when setting up deployment credentials for automated configuration engines (like Ansible).

### What you want to achieve

SSH keys are uploaded to NICo and injected into provisioned machines during OS deployment. A **Key Group** bundles multiple keys so they can be referenced as a unit when creating an Instance.

### Add an SSH key

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/sshkey" \
  -d '{
    "name": "alice-laptop",
    "publicKey": "ssh-ed25519 AAAA... alice@laptop"
  }' | jq .

export SSH_KEY_ID="<id from response>"
```

```shell
nicocli sshkey create --name alice-laptop \
  --public-key "ssh-ed25519 AAAA... alice@laptop"
```

### Create a key group

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/sshkeygroup" \
  -d '{
    "name": "team-keys",
    "sshKeyIds": ["'"$SSH_KEY_ID"'"]
  }' | jq .

export SSH_KEY_GROUP_ID="<id from response>"
```

```shell
nicocli sshkeygroup create --name team-keys --ssh-key-ids "$SSH_KEY_ID"
```

### List keys and groups

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/sshkey" | jq .

curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/sshkeygroup" | jq .
```

---

## 

## Flow 11 — Operating System Registration

### What it does
Registers base operating system images (ISO, QCOW2) or configures network booting schemas (raw/templated iPXE files containing preseed/kickstart installation directives).

### When to invoke
When updating baseline operating systems, security patches, or creating specialized machine images for AI workloads.

### What you want to achieve

NICo provisions machines by booting them via iPXE. Before creating an Instance you must register an OS record describing how to boot and install the OS. Three types are supported:

| Type | When to use |
| :---- | :---- |
| `Image` | Pre-built disk image with URL \+ SHA checksum |
| `iPXE` | Raw iPXE script uploaded directly |
| `Templated iPXE` | iPXE script template that NICo parameterises per machine |

### Register an OS (image type, Tenant)

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/operating-system" \
  -d '{
    "name": "ubuntu-22.04-server",
    "description": "Ubuntu 22.04 LTS cloud image",
    "tenantId": "'"$TENANT_ID"'",
    "siteIds": ["'"$SITE_ID"'"],
    "imageUrl": "https://images.example.com/ubuntu-22.04-server.qcow2",
    "imageSha": "sha256:abcdef1234...",
    "imageAuthType": "none"
  }' | jq .

export OS_ID="<id from response>"
```

```shell
nicocli operating-system create \
  --name ubuntu-22.04-server \
  --image-url "https://images.example.com/ubuntu-22.04-server.qcow2" \
  --image-sha "sha256:abcdef1234..."
```

### Register an OS (raw iPXE, Tenant)

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/operating-system" \
  -d '{
    "name": "rhcos-4.16-ipxe",
    "description": "Red Hat CoreOS 4.16 via iPXE",
    "tenantId": "'"$TENANT_ID"'",
    "siteIds": ["'"$SITE_ID"'"],
    "ipxeScript": "#!ipxe\nkernel http://boot.example.com/rhcos.kernel\ninitrd http://boot.example.com/rhcos.initrd\nboot"
  }' | jq .
```

### List operating systems

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/operating-system" | jq .

nicocli operating-system list
```

---

## 

## Flow 12 — Create & Manage Instances (with Networking)

### What it does
Spins up a live end-user environment on bare-metal by assigning a clean network segment (VPC/Subnet), pulling an OS image, deploying it onto an allocated machine, and tracking its operational state.

### When to invoke
Whenever an end-user or pipeline requests a brand-new clean node for training, or tears down an environment upon completion.

### What you want to achieve

An **Instance** binds a machine to a Tenant workload. NICo provisions the OS, injects SSH keys, and connects the machine to the Tenant's VPC. Before creating an Instance you need a VPC and (optionally) a subnet.

### Create a VPC

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/vpc" \
  -d '{
    "name": "training-vpc",
    "description": "VPC for AI training workloads"
  }' | jq .

export VPC_ID="<id from response>"
```

```shell
nicocli vpc create --name training-vpc
```

### Create an IP block (Provider) and subnet (Tenant)

The Provider first creates an IP block at the site, then the Tenant creates subnets within IP blocks allocated to them.

```shell
# Provider: create an IP block
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/ipblock" \
  -d '{
    "name": "amsterdam-pool-v4",
    "siteId": "'"$SITE_ID"'",
    "prefix": "10.100.0.0",
    "prefixLength": 16,
    "protocolVersion": "IPv4",
    "routingType": "bgp"
  }' | jq .

export IP_BLOCK_ID="<id from response>"

# Tenant: create a subnet within the allocated IP block
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/subnet" \
  -d '{
    "name": "training-subnet",
    "vpcId": "'"$VPC_ID"'",
    "ipv4BlockId": "'"$IP_BLOCK_ID"'",
    "prefixLength": 24
  }' | jq .
```

### Create an instance

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/instance" \
  -d '{
    "name": "training-node-01",
    "description": "H100 training workload",
    "tenantId": "'"$TENANT_ID"'",
    "instanceTypeId": "'"$INSTANCE_TYPE_ID"'",
    "vpcId": "'"$VPC_ID"'",
    "operatingSystemId": "'"$OS_ID"'",
    "sshKeyGroupId": "'"$SSH_KEY_GROUP_ID"'",
    "userData": "#!/bin/bash\napt-get update && apt-get install -y cuda-toolkit-12-6"
  }' | jq .

export INSTANCE_ID="<id from response>"
```

```shell
nicocli instance create \
  --name training-node-01 \
  --instance-type-id "$INSTANCE_TYPE_ID" \
  --vpc-id "$VPC_ID" \
  --operating-system-id "$OS_ID" \
  --ssh-key-group-id "$SSH_KEY_GROUP_ID"
```

### List and inspect instances

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/instance" | jq .

curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/instance/$INSTANCE_ID" | jq .

# Network interfaces attached to the instance
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/instance/$INSTANCE_ID/interface" | jq .

# Instance status history
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/instance/$INSTANCE_ID/status-history" | jq .
```

```shell
nicocli instance list
nicocli instance get "$INSTANCE_ID"
```

### Delete an instance

```shell
curl -sk -X DELETE -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/instance/$INSTANCE_ID"
```

---

## Flow 13 — Declarative Bootstrap with nicocli

### What it does
Applies an entire infrastructure state (Site parameters, SKUs, Network templates, Allocations) monolithically via a single YAML or JSON declarative file using the nicocli apply mechanic.

### When to invoke
When treating your infrastructure as code (IaC), spinning up standard replica data center zones, or restoring site state from a Git repository.

### What you want to achieve

Instead of calling individual API endpoints, `nicocli site bootstrap` applies a declarative YAML manifest that creates and verifies all prerequisite resources in the correct order: sites, IP blocks, VPCs, VPC prefixes, instance types, allocations, and instances. Useful for repeatable environment setup and CI pipelines.

```shell
nicocli site bootstrap --file site-manifest.yaml
```

Example manifest structure:

```
sites:
  - name: dc-amsterdam-01
    description: Primary Amsterdam DC

ipBlocks:
  - name: amsterdam-pool-v4
    siteRef: dc-amsterdam-01
    prefix: 10.100.0.0
    prefixLength: 16
    protocolVersion: IPv4
    routingType: bgp

instanceTypes:
  - name: h100-8gpu
    siteRef: dc-amsterdam-01
    controllerMachineType: server

allocations:
  - name: amsterdam-h100-pool
    siteRef: dc-amsterdam-01
    instanceTypeRef: h100-8gpu
    count: 4

vpcs:
  - name: training-vpc

instances:
  - name: training-node-01
    instanceTypeRef: h100-8gpu
    vpcRef: training-vpc
    operatingSystemRef: ubuntu-22.04-server
    sshKeyGroupRef: team-keys
```

---

## 

## Flow 14 — Audit Log

### What it does
Queries the cluster's immutable ledger to pull cryptographically verified administrative event records (who invoked what API, when, and from which IP).

### When to invoke
For security compliance reporting, post-mortem failure analysis, or tracing unexpected hardware reboots/reconfigurations.

### What you want to achieve

The audit log records every mutating API call — who made it, what endpoint, what the request body was, and the HTTP status code returned. Use it to trace changes, diagnose issues, and satisfy compliance requirements.

```shell
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/audit" | jq .
```

Example entry (from cluster):

```json
{
  "id": "446247c3-8954-4083-b8ec-1b5222890a58",
  "endpoint": "/v2/org/ncx/nico/site",
  "method": "POST",
  "body": {
    "description": "Managed by Helm",
    "name": "my-site"
  },
  "statusCode": 201,
  "clientIP": "10.2.32.116",
  "user": {
    "firstName": "ncx-service",
    "email": ""
  }
}
```

```shell
nicocli audit list

# Get a specific audit entry
nicocli audit get <auditEntryId>
```

---

## 

## Flow 15 — BMC Credential Management

### What it does
Updates, applies, and rotates IPMI/Redfish passwords and certificates across the entire local fleet or on specific targets to avoid default or stale access states.

### When to invoke
During periodic security compliance windows (e.g., rotating passwords every 90 days) or immediately following an engineer offboarding event.

### What you want to achieve

NICo connects to machine BMCs to control power, collect inventory, and run provisioning workflows. The **BMC credential** flow lets the Provider set or rotate the BMC password — either site-wide (all machines at a site share the new password) or per-machine (targeted by MAC address).

### Set or update BMC credentials

```shell
# Site-wide credential (applies to all machines at the site)
curl -sk -X PUT -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/credential/bmc" \
  -d '{
    "siteId": "'"$SITE_ID"'",
    "kind": "site",
    "username": "root",
    "password": "new-secure-password"
  }' | jq .

# Per-machine credential (targeted by BMC MAC address)
curl -sk -X PUT -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/credential/bmc" \
  -d '{
    "siteId": "'"$SITE_ID"'",
    "kind": "machine",
    "macAddress": "aa:bb:cc:dd:ee:01",
    "username": "root",
    "password": "machine-specific-password"
  }' | jq .
```

### Trigger a credential rotation

NICo can rotate credentials across all machines at a site asynchronously:

```shell
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$API_URL/v2/org/$ORG/nico/credential/rotation" \
  -d '{"siteId": "'"$SITE_ID"'"}' | jq .

# Check rotation status
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$API_URL/v2/org/$ORG/nico/credential/rotation" | jq .
```

---

## Quick Reference — Variable Dependency Map

```
Flow 1  → TOKEN
Flow 2  → INFRA_PROVIDER_ID, TENANT_ID
Flow 3  → SITE_ID                         (needs INFRA_PROVIDER_ID)
Flow 7  → SKU_ID                          (needs SITE_ID)
Flow 4  → (pre-register)                  (needs SITE_ID, SKU_ID)
Flow 5  → MACHINE_ID                      (needs SITE_ID)
Flow 8  → INSTANCE_TYPE_ID               (needs SITE_ID, MACHINE_ID)
Flow 9  → ALLOCATION_ID                  (needs TENANT_ID, INSTANCE_TYPE_ID)
Flow 10 → SSH_KEY_ID, SSH_KEY_GROUP_ID
Flow 11 → OS_ID                           (needs TENANT_ID, SITE_ID)
Flow 12 → VPC_ID, INSTANCE_ID            (needs all above)
```

# 

# Notions Vocabulary

In this chapter listed main notions including their definitions and relations to other notions from NICo:

🏢 Platform Multi-Tenancy & Identity

Infra Provider (Provider)

* **Definition:** The root administrative entity or infrastructure operator that owns, provisions, and manages the physical hardware, sites, and global configurations.
* **Relations:**
    * **To Tenants:** A single Provider manages and carves out isolated capacity for **multiple Tenants**.
    * **To Instance Types & SKUs:** The Provider defines global **SKUs** and **Instance Types** that are made available to tenants.
    * **To Allocations:** The Provider creates **Allocations** to assign resources to specific Tenants.

Tenant

* **Definition:** An isolated business unit, project, or customer entity that consumes infrastructure capacity allocated by the Infra Provider.
* **Relations:**
    * **To Sites:** **A single Site can serve multiple Tenants simultaneously** via network isolation and dedicated physical nodes.
    * **To Instances:** A Tenant owns **Instances**, **VPCs**, **Subnets**, and **SSH Keys**. They cannot see or access other tenants' resources.
    * **To Machines/DPUs:** **A single physical machine cannot usually be split among multiple tenants at the bare-metal level**, but multiple DPUs inside a single chassis can be isolated. However, NICo typically assigns a whole **Machine** to a single tenant instance for hard multi-tenant isolation.

Keycloak & nicocli

* **Definition:** **Keycloak** is the identity provider handling authentication via OAuth2 (specifically Machine-to-Machine client\_credentials). **nicocli** is the command-line interface tool used by both Providers and Tenants to interact with the NICo API.
* **Relations:**
    * **To Flows:** Used in nicocli login to obtain a token, enforcing Role-Based Access Control (RBAC) separating Provider actions from Tenant actions.

---

📍 Physical Infrastructure Topology

Site (site-id)

* **Definition:** A physical data center location, edge location, or room where a distinct cluster of hardware is deployed.
* **Relations:**
    * **To Provider/Tenant:** Owned by the Provider, but hosts infrastructure that serves multiple Tenants.
    * **To Machines:** A Site contains many **Expected Machines** and **Discovered Machines**.
    * **To Site Bootstrap:** Sites use a pairing/bootstrap lifecycle to sync local agents with the global NICo controller.

Machine (Expected vs. Discovered)

* **Definition:** A physical bare-metal server chassis containing CPUs, GPUs, network interfaces, a Baseboard Management Controller (BMC), and DPUs.
    * **Expected Machine:** A pre-registered skeleton entry (MAC addresses, serials) added *before* hardware is racked.
    * **Discovered Machine:** A live hardware entity found automatically via Redfish/LLDP when powered on.
* **Relations:**
    * **To DPUs/GPUs:** A Machine physical hosts these sub-components and exposes their operational telemetry.
    * **To Tenants:** When unassigned, it belongs to the Provider pool. When deployed, it is completely bound to a single Tenant's **Instance**.

SKU (Stock Keeping Unit)

* **Definition:** A hardware profile template that defines a specific combination of CPU, Memory, GPU, and DPU layout (e.g., H100-8GPU-DualDPU).
* **Relations:**
    * **To Machine:** Every **Discovered Machine** is matched against a **SKU** to automatically classify its capabilities.
    * **To Instance Types:** SKUs form the physical backing requirement for an **Instance Type**.

---

⚙️ Hardware Management & Controls

BMC (Baseboard Management Controller)

* **Definition:** A dedicated service processor on the motherboard that allows remote, out-of-band management (Power, Reset, Boot control) via protocols like Redfish.
* **Relations:**
    * **To Machine:** Every **Machine** has a BMC endpoint.
    * **To BMC Credentials:** Managed globally per-site or rotated per-machine for zero-trust security.
    * **To Power & BMC Reset:** Targeted by administrative actions to reboot or recover frozen hardware.

acknowledgeAttachedInstance Safety Flag

* **Definition:** A critical safety boolean parameter used during destructive out-of-band actions (like a hard power-down or BMC reset).
* **Relations:**
    * **To Machine & Instance:** If a **Machine** has an active, running Tenant **Instance** attached to it, NICo will block a power reset *unless* this flag is explicitly set to true. This prevents accidental disruption of production tenant workloads.

---

🏎️ Logical Cloud Layer (The Consumption Model)

Instance Type

* **Definition:** A logical flavor or sizing tier (similar to AWS instance sizes like p4d.24xlarge) created by the Provider that abstracts physical SKUs.
* **Relations:**
    * **To SKU:** Maps directly to one or more physical **SKUs**.
    * **To Allocation:** The currency used when a Provider carves out capacity (e.g., "Tenant A gets 10 of Instance Type X").

Allocation

* **Definition:** A quota or reservation certificate granting a specific Tenant the right to deploy a set number of **Instance Types** within a specific **Site**.
* **Relations:**
    * **To Tenant & Provider:** Created by the Provider; consumed by the Tenant.
    * **To Instance:** A Tenant must have an active **Allocation** balance to launch a new **Instance**.

Instance

* **Definition:** A living, provisioned bare-metal deployment requested by a Tenant. It is the real-world manifestation of an Instance Type.
* **Relations:**
    * **To Machine:** An Instance claims and binds to exactly one physical **Machine** from the available pool.
    * **To Tenant:** Wholly owned and controlled by the Tenant.

---

🌐 Networking & Security Isolation

VPC & Subnet

* **Definition:** **VPC (Virtual Private Cloud)** is an isolated tenant-specific logical network descriptor. **Subnet** is a segmented IP block inside that VPC.
* **Relations:**
    * **To Instance:** An **Instance** cannot be created without being attached to a **Subnet** inside a **VPC**.
    * **To DPU:** The DPU offloads and enforces this VPC encapsulation (BGP EVPN, VXLAN) in hardware so that the physical host machine's OS doesn't even realize it's sharing a network plane with other tenants.

SSH Keys & Key Groups

* **Definition:** Cryptographic public credentials injected into the bare-metal OS during provisioning to allow secure, passwordless access.
* **Relations:**
    * **To Instance:** Tied to the **Instance** creation manifest to ensure the Tenant can log in post-deployment.

---

💿 OS Provisioning & Observability

OS Registration (Image / iPXE)

* **Definition:** The definition of the operating system lifecycle delivery mechanism. It can be a static golden **Image**, a raw **iPXE network boot script**, or a **Templated iPXE** definition that dynamically alters boot parameters based on the machine ID.
* **Relations:**
    * **To Instance:** The Tenant selects a registered OS profile when launching an **Instance**, triggering the NICo deployment engine to stream it to the bare-metal host.

Audit Log

* **Definition:** A chronological, immutable record of API calls, system state alterations, and security events across the entire live cluster.
* **Relations:**
    * **To All Notions:** Captures who (Provider or Tenant via Keycloak identity) did what (created an Instance, rotated BMC keys, or overrode a safety flag) to any entity in the ecosystem.