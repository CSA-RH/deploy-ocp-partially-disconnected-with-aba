# OpenShift SNO - Partially Disconnected Installation with ABA

## Introduction
The objective of this repository is to do an exercise to showcase the ABA tool.  Here will do an exercise to install a Partially disconnected OpenShift cluster and deploy a local minimal Quay registry, using the mirror-registry tool.

The OpenShift cluster doesnt have internet access while the bastion host has network access to internet and to the OpenShift cluster.

The platform images will be pulled directly from the public redhat registries to the local Quay registry.

> **Note on platform choice:** > ABA supports `platform=kvm` which fully automates VM creation, boot, and lifecycle management. But here we use `platform=bm` (bare-metal) intentionally to simulate all the steps required to create a VM in KVM and start the installation of a SNO OpenShift cluster using a Agent Based Installer method.

~!!!!!!!!!!!!!!!!disclaimer
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

## Environment Summary


| Parameter                       | Value                                                      |
| ------------------------------- | ---------------------------------------------------------- |
| Base domain                     | `ocp4.home.levmdomain.com`                                 |
| Cluster name                    | `sno1`                                                     |
| Cluster type                    | SNO (Single Node OpenShift)                                |
| Network CIDR                    | `192.168.1.0/24`                                           |
| SNO node IP                     | `192.168.1.245`                                            |
| Bastion (RHEL 9.7 KVM VM)       | `registry.home.levmdomain.com` → `192.168.1.152` (bastion) |
| Gateway                         | `192.168.1.1` (adjust to your environment)                 |
| DNS server                      | `192.168.1.1` (adjust to your environment)                 |
| NTP server                      | `192.168.1.1` (adjust to your environment)                 |
| Operators to mirror             | `openshift-gitops-operator`, `cincinnati-operator`         |
| OCP version to mirror 4.21.3/10 |                                                            |
| Deployment mode                 | Partially disconnected (only bastion has Internet)         |

---

## DNS Records Configured

These DNS records must exist **before** starting the OpenShift install.
Configure them on your DNS server (e.g. dnsmasq, BIND, Pi-hole, router DNS).


| Record                                 | Type | Value           |
| -------------------------------------- | ---- | --------------- |
| `api.sno1.ocp4.home.levmdomain.com`    | A    | `192.168.1.245` |
| `*.apps.sno1.ocp4.home.levmdomain.com` | A    | `192.168.1.245` |
| `registry.home.levmdomain.com`         | A    | `192.168.1.152` |

---

## Part 1 — RHEL 9.7 Bastion VM (KVM)

### 1.1 Create the Bastion host, that will also contain the minimal Quay registry

- Download the RHEL 9.7 ISO

- On your KVM hypervisor host, create the VM:

    ```bash
    sudo virt-install \
      --name bastion-ocp \
      --ram 16384 \
      --vcpus 4 \
      --disk path=/var/lib/libvirt/images/bastion-ocp.qcow2,size=200,format=qcow2 \
      --os-variant rhel9.7 \
      --network bridge=br0 \
      --console pty,target_type=serial \
      --cdrom /home/luis/Downloads/rhel-9.7-x86_64-dvd.iso \
      --boot uefi
    ```

### 1.2 Configure Static Network on the Bastion host

After the OS installation, check that OS network is correctly configured

```bash
nmcli device status
ip addr show
ip route show
cat /etc/resolv.conf
```

**If the network is correctly configured, skip to 1.3 Register the system and install base packages.**
If the network needs to be configured or corrected, identify your interface name from the
`nmcli device status` output above (e.g. `enp1s0`, `eth0`, `ens3`) and run:

```bash
# Delete any existing DHCP connection (adjust name to match your system)
sudo nmcli connection delete "Wired connection 1" 2>/dev/null
sudo nmcli connection delete enp1s0 2>/dev/null

# Create a new static connection
sudo nmcli connection add \
  con-name static-bastion \
  type ethernet \
  ifname enp1s0 \
  ipv4.addresses 192.168.1.152/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "192.168.1.1" \
  ipv4.dns-search "registry.home.levmdomain.com" \
  ipv4.method manual \
  connection.autoconnect yes

# Activate the connection
sudo nmcli connection up static-bastion
```

Verify the configuration is now correct:

```bash
ip addr show enp1s0
ping -c 2 192.168.1.1
ping -c 2 8.8.8.8
nslookup registry.home.levmdomain.com
```

Set the hostname

```bash
sudo hostnamectl set-hostname registry.home.levmdomain.com
```

### 1.3 Register the system and install base packages

```bash
sudo subscription-manager register \
      --org="<or_number>" \
      --activationkey="<key_name>"

sudo subscription-manager repos \
      --enable=rhel-9-for-x86_64-baseos-rpms \
      --enable=rhel-9-for-x86_64-appstream-rpms 

#upgrade OS
sudo dnf update -y

#Install additional packages
sudo dnf install -y git make jq  firewalld jq tar net-tools bind-utils wget podman bash-completion unzip openssl python3 python3-jinja2 python3-pyyaml ncurses which diffutils dialog httpd-tools skopeo

sudo shutdown -r 0
```

### 1.5 Enable passwordless SSH to localhost

ABA and the Quay mirror-registry installer require passwordless SSH from the bastion to itself.

```bash
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
ssh localhost "hostname"   # should succeed without a password prompt
```

Also add root passwordless (not a good practice):

```bash
# First, authenticate with your password
sudo -i

# Then add the NOPASSWD rule using a drop-in file (safer than editing the main sudoers)
echo 'luis ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/luis
chmod 440 /etc/sudoers.d/luis

# Exit root
exit

# Test it
sudo whoami
```

### 1.6 Configure firewall

Open the ports needed for the Quay mirror registry and HTTP:

```bash
sudo firewall-cmd --permanent --add-port=22/tcp
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

### 1.7 Download the pull secret

Download your Red Hat pull secret from [https://console.redhat.com/openshift/install/pull-secret](https://console.redhat.com/openshift/install/pull-secret)
and save it:

```bash
vi ~/.pull-secret.json
# Paste the pull secret content and save
```

---

## Part 2 — Install and Configure ABA

### 2.1 Clone and install ABA

Find the latest stable release in **https://github.com/sjbylo/aba/releases**

Than clone the actual stable release and install `aba`:

```bash
cd ~
wget https://github.com/sjbylo/aba/archive/refs/tags/v1.0.0.tar.gz
tar xzf v1.0.0.tar.gz
cd aba-1.0.0
./install
```

Run aba to create and configure the aba.conf file:

```
aba          # Let ABA guide you through the OpenShift installation workflow (interactive mode)
```

output
```
[luis@registry aba-1.0.0]$ ./install
Installing required rpms: coreos-installer httpd nmstate (logging to .dnf-install.log). Please wait!
ABA has been installed to /home/luis/bin/aba
[luis@registry aba-1.0.0]$
```

### 2.2 Configure `aba.conf`

Run `aba` and edit the configuration when prompted, or edit `aba.conf` directly:

```bash
vi aba.conf
```

output: [aba config output](outputs/output1-aba_config.txt)

Configuration file:

```
[aba-1.0.0]$ cat aba.conf 
# Aba global configuration file

ocp_channel=stable		# OpenShift release channel ('stable', 'fast', 'candidate', or 'eus').
ocp_version=4.21.10		# Target OpenShift version. See https://console.redhat.com/openshift/releases for available versions.

platform=bm			# Deployment platform: 'vmw' (vCenter/ESXi), 'kvm' (KVM/libvirt), or 'bm' (bare-metal, skips VM creation).

# Set the following values only if they are known.

op_sets=			# Optional: Comma-separated operator sets, such as: op_sets=odf,virt,acm,mesh3).
				# Operator sets are defined under 'templates/operator-set-*'. Add or edit as needed.
				# Valid values: ocp,acm,virt,odf,appdev,mesh2,mesh3,odfdr,quay,sec,ai
				# 'all' includes every operator in redhat-operator-index (ensure sufficient disk space!).

ops=web-terminal,cincinnati-operator				# Optional: Comma-separated individual operator names such as: ops=web-terminal,devworkspace-operator
				# If needed, fetch operator names from the files: mirror/imageset-config-*.yaml, generate by: aba catalog

# *If you know then*, set the following important values for your disconnected network.
# When a cluster is created, these values will be included - and can be overridden - in the cluster’s cluster.conf file.

domain=ocp4.home.levmdomain.com		# Base domain for your disconnected OpenShift cluster.
machine_network=192.168.1.0/24	# Network segment (CIDR format) where OpenShift will be installed.
dns_servers=192.168.1.1		# DNS server IPs (comma-separated).
next_hop_address=192.168.1.1	# Default gateway IP for your private network.
ntp_servers=2.rhel.pool.ntp.org		# Important: NTP servers or hostnames (comma-separated). Needed if server clocks aren't synced.
				# Note: The above 5 values can be overridden later in 'cluster.conf'.

# Other settings

pull_secret_file=~/.pull-secret.json	# File path for your Red Hat pull secret, used for image downloads only.
					# Get it from: https://console.redhat.com/openshift/downloads -> Tokens -> Pull-secret.
editor=vi			# Text editor to use, such as: nano, vi, emacs. Set to 'none' for manual config editing.
ask=true			# Ask for confirmation before major actions (install, delete, etc.).

#######################################################################################
# ADVANCED SETTINGS — CHANGE ONLY IF REQUIRED

excl_platform=false		# Exclude platform images. Leave 'false' in most cases.

verify_conf=all			# Validate config files and run network checks (DNS, NTP, IP conflicts).
				# Set to 'conf' to validate config files only (skip network checks).
				# Set to 'off' to skip all validation.
```

---

## Part 3 — Mirror Registry Setup and Image Sync

This section breaks the mirror workflow into individual steps so you can understand
and verify each one before proceeding. ABA's `aba -d mirror sync` can do all of this
in a single command, but here we run each step separately.

### 3.1 Install the mirror registry

This installs the Mirror Registry for Red Hat OpenShift (Quay) on the bastion.
It will auto-create `mirror/mirror.conf` and prompt you to edit it before proceeding.

What this command does:

1. Initializes the `mirror/` directory (creates symlinks to `scripts/`, `templates/`, `cli/`)
2. Installs required RPMs on the bastion
3. Downloads and extracts the Quay mirror-registry installer
4. Installs Quay on the bastion, generating TLS certificates and registry credentials
5. Stores credentials in `~/.aba/mirror/` and marks the registry as available

```bash
aba -d mirror install
```

output: [mirror registry install output](outputs/output2-install-mirror-registry.txt)

### 3.2 Verify the mirror registry

Confirm the Quay registry is running and credentials are working before syncing images:

```bash
aba -d mirror verify
```

### 3.3 Sync images from the Internet to the mirror registry

This pulls all required container images from Red Hat registries and pushes them into your local Quay mirror. Since the registry is already installed (step 3.1), this step skips the install and goes straight to mirroring.

What this command does:

1. Downloads operator catalog indexes (to resolve operator names and dependencies)
2. Generates `mirror/data/imageset-config.yaml` including:
   - OpenShift platform images for the configured version
   - Operator images for operators defined in `aba.conf` (`ops=` and `op_sets=`)
3. Runs `oc-mirror` to pull images from Red Hat registries and push them directly into `registry.home.levmdomain.com:8443`

The `--retry` flag is recommended because `oc-mirror` can fail on transient network issues — it will automatically retry the sync that many times.

```bash
aba -d mirror sync --retry 3
```

> **Note:** This step can take a long time (30 minutes to several hours) depending
> on your Internet speed and the number of operators configured.

output: [sync images mirror registry](outputs/output3-sync-images-with-mirrorregistry.txt)

The `mirror/data/working-dir/cluster-resources/` directory is created by oc-mirror every time you run `aba -d mirror sync` or `aba -d mirror load`. It contains YAML manifests that `aba day2` applies to the cluster. The files are:

- idms-oc-mirror.yaml (ImageDigestMirrorSet): redirects image pulls by digest from public registries to your local mirror.
- itms-oc-mirror.yaml (ImageTagMirrorSet): same redirection but for tag-based image references
- cs-*-index*.yaml (CatalogSource): tells OperatorHub where to find the mirrored operator catalog indexes, so operators appear in the web console.
- signature-configmap.json (ConfigMap): contains release image signatures so the cluster can verify the authenticity of mirrored OpenShift images.

Without these applied, the cluster would try to pull from public registries (fails in disconnected), OperatorHub would be empty, and release verification would fail. That's why aba day2 is required after every sync or load.

### 3.4 Verify the mirrored content

After the sync completes, verify that images were pushed to the registry:

```bash
aba -d mirror verify
```

You can also inspect the imageset config that was generated:

```bash
cat mirror/data/imageset-config.yaml
```

### 3.5 (Optional) Pre-download CLI binaries

If you plan to disconnect from the Internet before installing the cluster:

```bash
aba -d cli download
```

### 3.6. Minimal Quay access
The minimal Quay can be accessed in https://registry.home.levmdomain.com:8443

```
$ podman login -u init -p p4ssw0rd  https://registry.home.levmdomain.com:8443
Login Succeeded!
```

---

## Part 4 — Create and Install the OpenShift SNO Cluster


### 4.0. Force a refresh to download the correct binaries versions, e.g. fot install OpenShift version 4.21.3
ABA verifies that the `openshift-install` binary matches `ocp_version`:

```bash
aba -d cli clean install
```

### 4.1 Create the cluster directory

```bash
cd ~/aba-1.0.0/
aba cluster --name sno1 --type sno --starting-ip 192.168.1.245
```

### 4.2 Review `sno1/cluster.conf`

Verify the generated configuration:

```bash
cd sno1
cat cluster.conf
```

output: [generate ocp install files](outputs/output4-ocp-generate-install-files.txt)

Expected values for SNO:

```
cluster_name=sno1
base_domain=ocp4.home.levmdomain.com

api_vip=
ingress_vip=

machine_network=192.168.1.0/24

starting_ip=192.168.1.245

num_masters=1
num_workers=0

dns_servers=192.168.1.1
next_hop_address=192.168.1.1
ntp_servers=192.168.1.1

int_connection=

mirror_name=mirror

data_disk=120
```

> For SNO, `api_vip` and `ingress_vip` are ignored (both API and ingress use the node IP).
> `int_connection=` is empty because only the bastion has Internet access — the cluster
> uses the local mirror exclusively.

A new path with the name of the cluster `sno1` is created and contains the cluster parameters:

```bash
ls ~/aba-1.0.0/sno1/
```

### 4.3 Generate agent configuration files

```bash
aba agentconf
```

output: [generate ocp agent config](outputs/output5-ocp-generate-ocp-agent-config.txt)

This generates `install-config.yaml` and `agent-config.yaml`.

### 4.4 Edit agent-config.yaml

You must set the correct MAC address of the target node:

```bash
vi agent-config.yaml
```

Find the `interfaces` section and update the MAC address to match your KVM VM NIC:

```yaml
    interfaces:
      - name: enp1s0
        macAddress: 52:54:00:ab:bb:ce    # <-- replace with actual MAC
```

### 4.5 Generate the ISO

```bash
aba iso
```

output: [generate iso](outputs/output6-ocp-generate-iso.txt)

Lets copy the generated ISO to the workstation hosting KVM:

```
scp /home/luis/aba-1.0.0/sno1/iso-agent-based/agent.x86_64.iso luis@192.168.1.102:/home/luis/Downloads
```

### 4.6 Create the OpenShift SNO VM and boot from the ISO

Since we are simulating bare-metal on KVM, create the VM manually:

```bash
sudo virt-install \
  --name sno1 \
  --ram 64000 \
  --vcpus 8 \
  --disk path=/var/lib/libvirt/images/sno1.qcow2,size=120,format=qcow2 \
  --os-variant rhel9.7 \
  --network bridge=br0,mac=52:54:00:ab:bb:ce \
  --console pty,target_type=serial \
  --cdrom /home/luis/Downloads/agent.x86_64.iso \
  --boot uefi
```

### 4.7 Monitor the installation

Monitor the installation:

```bash
aba mon
```

output: [monitor openshift installation](outputs/output7-ocp-installation-monitor.txt)

---

## Part 5 — Day-2 Configuration (Post-Install)

### 5.1 Access the cluster - using KUBECONFIG

```bash
cd ~/aba-1.0.0/sno1
. <(aba shell)
oc whoami
oc get co
```

Wait until all cluster operators show `Available=True`.

### 5.2 Connect OperatorHub to the internal mirror (REQUIRED)

The `install-config.yaml` already includes the mirror registry CA, pull secret, and
`ImageDigestSources` for the OpenShift **platform release images** — that's how the
cluster installs successfully from the mirror. However, the cluster still does not know
about the mirrored **operator catalogs** or the broader image redirections for operator
images. Without this step, OperatorHub will be empty.

The `aba day2` command applies the manifests generated by `oc-mirror` during the sync
(stored in `mirror/data/working-dir/cluster-resources/`) to complete the configuration:

- **IDMS** (ImageDigestMirrorSet) — extends image pull redirections to cover all mirrored content (operators, not just platform images)
- **ITMS** (ImageTagMirrorSet) — same for tag-based references
- **CatalogSources** — configures OperatorHub to use the mirrored `redhat-operator-index` catalog, making operators visible and installable
- **Registry CA ConfigMap** — ensures the cluster-wide trust store includes the mirror registry's CA
- **Release signature ConfigMap** — validates the authenticity of OCP release images

> **CRITICAL:** You MUST re-run `aba day2` every time you run `aba -d mirror sync`
> on a cluster that is already running.

```bash
aba day2
```
output: [update openshift images](outputs/output8-ocp-update-images.txt)


### 5.3 Configure NTP
Creates machineconfig containing NTP server configuration

```bash
aba day2-ntp
```

Creates MachineConfig objects via Butane to configure chrony (NTP) on the SNO node
using the NTP servers defined in `aba.conf`. Verifies synchronisation via SSH.

### 5.4 Enable OpenShift Update Service

Configures OpenShift to receive updates via your internal mirror. Only required if OpenShift is fully disconnected, and used for clusters to be aware about the upgrades path. 
NOTE: The cincinnati-operator must be available in the mirror for OSUS to work!

```bash
aba day2-osus
```

Check that the cincinnati-operator it is feeding the versions
```
cd ~/aba-1.0.0/sno1
. <(aba shell)
curl -k https://osus-route-openshift-update-service.apps.sno1.ocp4.home.levmdomain.com/api/upgrades_info/v1/graph?channel=stable-4.21
```

output:
```
$ cd ~/aba-1.0.0/sno1
. <(aba shell)
curl -k https://osus-route-openshift-update-service.apps.sno1.ocp4.home.levmdomain.com/api/upgrades_info/v1/graph?channel=stable-4.21
{"version":1,"nodes":[{"version":"4.21.10","payload":"registry.home.levmdomain.com:8443/ocp4/openshift4/openshift/release-images@sha256:5d591a70c92a6dfa3b6b948ffe5e5eac7ab339c49005744006aa0dd9d6d98898","metadata":{"io.openshift.upgrades.graph.previous.remove_regex":".*","io.openshift.upgrades.graph.release.channels":"candidate-4.21,fast-4.21,stable-4.21,candidate-4.22","io.openshift.upgrades.graph.release.manifestref":"sha256:5d591a70c92a6dfa3b6b948ffe5e5eac7ab339c49005744006aa0dd9d6d98898","url":"https://access.redhat.com/errata/RHSA-2026:7245"}}],"edges":[],"conditionalEdges":[]}
```

---

## Part 6 — Adding new operators after the initial OpenShift cluster install

This section walks through a example of adding a new operator to your disconnected environment.

### 6.1 Find the correct operator name

Operator names must match the package name in the Red Hat operator catalog exactly. ABA downloads the catalog indexes during the mirror sync and stores them under `.index/` in the mirror directory. You can browse them to find the correct name:

```bash
cd ~/aba-1.0.0

# List all available operators in the Red Hat catalog for your OCP version
cat mirror/.index/redhat-operator-index-v4.21

# Search for a specific operator (e.g. logging)
grep -i logging mirror/.index/redhat-operator-index-v4.21

# Other catalogs are also available
cat mirror/.index/certified-operator-index-v4.21
cat mirror/.index/community-operator-index-v4.21
```

Each line in the index file shows: `<operator-name>  <display-name>  <default-channel>`

For example, to find the OpenShift Logging operator:

```bash
grep -i logging mirror/.index/redhat-operator-index-v4.21
```

Might return:

```
cluster-logging                 Red Hat OpenShift Logging               stable-6.5
```

The value you need for `aba.conf` is the first column: `cluster-logging`.

> **Tip:** ABA also provides pre-built operator sets in `templates/operator-set-*`.
> For example, `templates/operator-set-ocp` includes common operations operators
> (web-terminal, nmstate, descheduler, etc.). You can use these via `op_sets=ocp`
> in `aba.conf` instead of listing individual operators.

### 6.2 Configure the operator in aba.conf

Add the operator name to the `ops=` line in `aba.conf` (comma-separated, no spaces):
For example, to add `cluster-logging` alongside the existing operators:

```bash
cd ~/aba-1.0.0
vi aba.conf

ops=web-terminal,cincinnati-operator,cluster-logging
```

### 6.3 Re-sync the mirror

Re-running sync regenerates the `imageset-config.yaml` (since `aba.conf` changed) and mirrors the new operator images, to the minimal Quay registry:

```bash
aba -d mirror sync --retry 3
```

Verify the mirrored content:

```bash
# Check the imageset config that was generated
cat ~/aba-1.0.0/mirror/data/imageset-config.yaml
```

### 6.4 Update the cluster

From your cluster directory, re-run `day2` so the cluster sees the new operator
in OperatorHub:

If you have already installed a cluster, (re-)run the command to configure/refresh OperatorHub/Catalogs, Signatures etc. 

```bash
cd sno1
aba -d sno1 day2
```

### 6.5 Check that cluster-logging operator is available to be installed in the OpenShift catalogue

```bash
# Verify the operator is available in OperatorHub
oc get packagemanifest | grep cluster-logging
```



## Part 7 - Upgrading OpenShift

### Prerequisites for upgrading

Before you can upgrade OpenShift in a disconnected environment, the following must be in place:

1. **OpenShift Update Service (OSUS) must be enabled.** If you followed Part 5 and ran
   `aba day2-osus`, this is already done. OSUS allows the cluster to discover available
   upgrades via your internal mirror instead of reaching the Internet. This is only obligatory when the OpenShift cluster doesnt have internet access.

2. **The `cincinnati-operator` must be mirrored.** This operator powers OSUS.
   If you configured `ops=cincinnati-operator` in `aba.conf` (as in this guide), it is
   already in your mirror.

3. **The target OpenShift version images must be in your mirror registry.** ABA does not
   automatically mirror upgrade images — you must explicitly add the target version to
   the imageset config and re-sync.

### 7.1 Edit the imageset config to include the target version

The imageset config controls which OpenShift versions and operators are mirrored.
Open it and adjust the `minVersion` and `maxVersion` under `platform.channels`:

```bash
vi mirror/data/imageset-config.yaml
```

!!!!!!!!<continue here>
Find the `channels` section and update it. For example, to upgrade from 4.21.10 to 4.21.??:

```yaml
  platform:
    channels:
    - name: 'stable-4.21'
      minVersion: '4.21.3'
      maxVersion: '4.21.10'
      type: ocp
      shortestPath: true    # will only download the images required to upgrade from 4.21.3 to 4.21.10

```

> **Important:** ABA does **not** manage the `minVersion`/`maxVersion` values automatically.
> You must edit them manually before mirroring upgrade images.

> **Note on EUS upgrades:** For Extended Update Support (EUS-to-EUS) upgrades
> (e.g. 4.14 to 4.16), you must include intermediate versions in the channel range.
> Consult the [Red Hat upgrade documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/updating_clusters/index) for your specific upgrade path.

### 7.2 Sync the new images to the mirror registry

Re-run the sync to pull the new version images into your Quay mirror:

```bash
aba -d mirror sync --retry 3
```

This will download only the new/changed images (oc-mirror handles incremental mirroring).

### 7.3 Update the cluster's mirror configuration

After syncing new images, the cluster needs to know about the updated content.
Run `day2` from your cluster directory:

```bash
cd ~/aba-1.0.0/sno1
aba day2
```

This refreshes the IDMS/ITMS, CatalogSources, and release signatures so the cluster
can see the newly mirrored version.

### 7.4 Trigger the upgrade

After `day2` completes and OSUS has had time to update the graph (allow ~10 minutes),
the new version should appear in the OpenShift web console:

1. Open the OpenShift web console: `https://console-openshift-console.apps.sno1.ocp4.home.levmdomain.com`
2. Navigate to **Administration → Cluster Settings**
3. The available upgrade should appear in the **Update channel** section
4. Click **Update** and select the target version

### 7.5 Upgrading operators (already mirrored)

Operator upgrades work differently from platform upgrades. When `oc-mirror` syncs, it mirrors the **full operator catalog index** for each operator you've configured. 
That catalog contains metadata for **all available versions** of each operator. This means that when you re-sync, any new operator versions that Red Hat has published since your last sync are automatically pulled into your mirror.

You do **not** need to edit `imageset-config.yaml` to upgrade an existing operator.

1. Re-sync the mirror to pull updated catalogs and any new operator images:

```bash
cd ~/aba-1.0.0
aba -d mirror sync --retry 3
```

2. Refresh the cluster's CatalogSources so it sees the updated catalog:

```bash
cd sno1
aba day2
```

3. The new operator version now appears in OperatorHub. Upgrade it from the OpenShift web console (**Operators → Installed Operators → select operator → Upgrade**) or via CLI:

```bash
# Check current operator version
oc get csv -n openshift-operators

# If the operator's install plan approval is set to "Automatic", the upgrade
# happens automatically after day2 refreshes the catalog.
# If set to "Manual", approve the pending install plan:
oc get installplan -n openshift-operators
oc patch installplan <plan-name> -n openshift-operators --type merge -p '{"spec":{"approved":true}}'
```

> **Summary of the difference:**
>
> - **Platform (OpenShift) upgrade** -- edit `imageset-config.yaml` required, you choose the exact target version range
> - **Operator upgrade (already mirrored)** -- no edit needed, oc-mirror pulls all versions from the catalog automatically
> - **Adding a new operator** -- edit `aba.conf` then re-sync, ABA regenerates the imageset config

---

## Part 8 — Pruning older images not in use from the mirror registry

Over time the mirror registry accumulates images from previous OpenShift versions or operators you no longer need. ABA does not include a built-in prune command — image deletion is handled directly by `oc-mirror v2` using a dedicated two-stage delete workflow.

In: https://access.redhat.com/solutions/7109213
https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/disconnected_environments/about-installing-oc-mirror-v2#oc-mirror-workflows-delete-v2_about-installing-oc-mirror-v2

> **Why two stages?** `oc-mirror v2` removed the automatic pruning that v1 had.
> Deletion now requires an explicit configuration file (`DeleteImageSetConfiguration`)
> to prevent accidental removal of images that running clusters depend on.

### 8.1 Create a DeleteImageSetConfiguration

Create a YAML file that specifies exactly which images to remove. Only include the
sections relevant to what you want to delete.

```bash
cd ~/aba-1.0.0/mirror/data
vi delete-imageset-config.yaml
```

Example — delete platform version 4.21.3 through 4.21.8 and the `cluster-logging`
operator:

```yaml
apiVersion: mirror.openshift.io/v1alpha2
kind: DeleteImageSetConfiguration
delete:
  platform:
    channels:
      - name: stable-4.21
        minVersion: '4.21.3'
        maxVersion: '4.21.8'
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.21
      packages:
        - name: cluster-logging
```

You can include any combination of `platform`, `operators`, and `additionalImages`.
Omit a section entirely if you don't need to delete that type.

### 8.2 Generate the delete plan (dry-run)

This creates a list of all manifests and blobs that would be removed, without
actually deleting anything:

```bash
oc-mirror delete --v2 \
  --config delete-imageset-config.yaml \
  --workspace file://. \
  --generate \
  docker://registry.home.levmdomain.com:8443/ocp4/openshift4
```

Review the generated file at `working-dir/delete/delete-images.yaml` before proceeding.

### 8.3 Execute the deletion

Once you have reviewed and are satisfied with the delete plan:

```bash
oc-mirror delete --v2 \
  --delete-yaml-file working-dir/delete/delete-images.yaml \
  docker://registry.home.levmdomain.com:8443/ocp4/openshift4
```

Add `--force-cache-delete` if you also want to purge the local workspace cache
(blobs under `working-dir/`).

### Important considerations

- **Never delete images for a version that a running cluster is currently using.**
  If your cluster runs 4.21.10, do not delete 4.21.10 platform images.
- **Quay garbage collection:** The delete command removes manifests from the registry.
  Unreferenced blobs (layers) remain on disk until Quay's internal garbage collection
  process cleans them up automatically.
- **v1-mirrored images:** If your images were originally mirrored with `oc-mirror v1`,
  use the `--delete-v1-images` flag instead of the standard delete workflow.

> **References:**
> - [oc-mirror v2 delete functionality](https://github.com/openshift/oc-mirror/blob/main/docs/features/delete-functionality.md)
> - [Red Hat OpenShift — oc-mirror v2 disconnected install](https://docs.openshift.com/container-platform/4.16/installing/disconnected_install/about-installing-oc-mirror-v2.html)
> - [Red Hat KB: deleting v1-mirrored images with v2](https://access.redhat.com/solutions/7109213)

---

## Part 10 — Deleting a cluster completely

Since `platform=bm` (bare-metal) was used, ABA did not create the VM and cannot
delete it automatically. The `aba delete` command only works for `platform=kvm` or
`platform=vmw` where ABA manages the VM lifecycle. For bare-metal, each step must
be done manually.

### 10.1 Clean up ABA cluster files

```bash
cd ~/aba-1.0.0/sno1
aba clean
cd ..
rm -rf sno1
```

`aba clean` removes generated files (ISO, agent configs, marker files) but preserves
`cluster.conf`. The `rm -rf` removes the entire cluster directory. There is no
persistent per-cluster state in `~/.aba/` — that directory only holds mirror
credentials and runner cache.

### 10.2 Destroy the KVM virtual machine

Since the VM was created manually with `virt-install`, it must be destroyed manually:

```bash
sudo virsh destroy sno1
sudo virsh undefine sno1 --remove-all-storage --nvram
```

`destroy` forces power-off. `undefine --remove-all-storage --nvram` removes the VM
definition, its disk (`/var/lib/libvirt/images/sno1.qcow2`), and the UEFI NVRAM file.

Verify it is gone:

```bash
sudo virsh list --all | grep sno1
```

### 10.5 Remove the ISO from the KVM host

The ISO was copied to the hypervisor during installation:

```bash
rm /home/luis/Downloads/agent.x86_64.iso   # on the KVM host (192.168.1.102)
```

### 10.6 (Optional) Prune mirror registry images

The sno1 images (4.21.10) remain in the mirror registry. If no other cluster uses
that version and you want to reclaim disk space, follow Part 8 (pruning). Otherwise
leave them — they cause no harm and `oc-mirror` handles deduplication for shared
layers.

