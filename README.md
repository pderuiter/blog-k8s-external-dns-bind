# blog-k8s-external-dns-bind

Companion code for the blog post
[Integrating Kubernetes External-DNS with BIND for Automatic DNS Management](https://webuildyourcloud.com/integrating-kubernetes-external-dns-with-bind-for-automatic-dns-management/).

It builds the setup from the post. It creates two Ubuntu 24.04 VMs:
- A BIND server that accepts RFC2136 updates, signed with a TSIG key, for `k8s.example.com`.
- A single-node k3s cluster running External-DNS v0.14.2 with the RFC2136 provider.

The cluster uses the post's ServiceAccount/ClusterRole, Deployment and egress NetworkPolicy. A LoadBalancer Service
with an `external-dns.alpha.kubernetes.io/hostname` annotation then gets an A record (plus the TXT ownership records)
in BIND, and the records are removed again when the Service goes away.

## What's in here

| Path | What it does |
| --- | --- |
| `terraform/` | Two Proxmox VE VMs from the Ubuntu 24.04 cloud image (bpg/proxmox provider): `dns` and `k8s`. Outputs an Ansible inventory. |
| `ansible/roles/bind_install` | Installs BIND (Ubuntu 24.04: 9.18; RHEL 9: 9.16). |
| `ansible/roles/bind_external_dns` | The post's BIND part. It runs `tsig-keygen -a hmac-sha256 external-dns-key` (key mode 0640, group `bind`/`named`), creates the `type primary` zone(s) with `update-policy { grant external-dns-key zonesub ANY; }` and the initial zone file, and can add the optional reverse zone and the `update_log` channel. Paths follow the OS: `/etc/named/keys` and `/var/named/dynamic` on RHEL, as in the post; `/etc/bind/keys` and `/var/lib/bind/dynamic` on Debian/Ubuntu. |
| `ansible/roles/k3s` | k3s v1.37.2+k3s1 (its bundled Traefik disabled), Helm v3.19.0, and a virtualenv with the kubernetes Python client for `kubernetes.core`. |
| `ansible/roles/external_dns` | Namespace, the `external-dns-tsig` Secret (the secret is read from the key file on the BIND server), RBAC, the NetworkPolicy (BIND on 53, the API server on 443/6443) and the Deployment. The secret is passed as `EXTERNAL_DNS_RFC2136_TSIG_SECRET`. |
| `ansible/site.yml` | BIND, then k3s, then External-DNS. |
| `tests/verify.yml` | The post's check: deploys `my-app` with a LoadBalancer Service for `myapp.k8s.example.com`, waits for the A record on BIND, deletes the Service and waits for the record to disappear. |

k3s's ServiceLB gives LoadBalancer Services the node's IP, so `status.loadBalancer.ingress` is filled in without
MetalLB or a cloud load balancer.

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VMs).
- Ansible >= 2.16 (ansible-core) and the collections in `ansible/requirements.yml`.
- SSH access with sudo to both machines. The Terraform cloud-init user is `ubuntu`.
- Internet access from the VMs (packages, get.k3s.io, get.helm.sh, registry.k8s.io).

## Quick start

```bash
# 1. Machines (optional; any two Ubuntu 24.04 hosts work)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in endpoint, token, node, IPs, SSH key
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Configuration
cd ../ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml
ansible-playbook ../tests/verify.yml    # optional end-to-end check
```

Without Terraform, copy `ansible/inventory.example.yml` to `ansible/inventory.yml` and put your hosts in the `dns`
and `k8s` groups. The TSIG key is generated on the BIND server and copied into the cluster by the playbook, so there
is no vault to fill in.

On the k3s node, `kubectl` is `k3s kubectl` (or `kubectl` with `KUBECONFIG=/etc/rancher/k3s/k3s.yaml`):

```bash
sudo k3s kubectl logs -n external-dns deployment/external-dns -f
```

## Variables

| Variable | Default | Meaning |
| --- | --- | --- |
| `external_dns_zone` | `k8s.example.com` | Zone in BIND, `--rfc2136-zone` and `--domain-filter` |
| `external_dns_tsig_keyname` | `external-dns-key` | TSIG key name |
| `external_dns_txt_owner_id` | `k8s-cluster` | `--txt-owner-id` |
| `external_dns_image` | `registry.k8s.io/external-dns/external-dns:v0.14.2` | External-DNS image |
| `external_dns_sources` | `[service, ingress]` | `--source` flags; add `gateway-httproute` for Gateway API |
| `external_dns_policy` | `sync` | `--policy` (`upsert-only` never deletes) |
| `external_dns_extra_args` | `[]` | Extra flags, e.g. `--txt-prefix=_edns-prod-` or `--annotation-filter=...` |
| `external_dns_rfc2136_host` | first host in `dns` | BIND server address |
| `external_dns_network_policy` | `true` | Apply the egress NetworkPolicy |
| `external_dns_api_server_cidrs` | the k3s node IP | Control-plane address(es) the policy allows |
| `bind_external_dns_update_policy` | `grant external-dns-key zonesub ANY;` | Use `subdomain k8s.example.com ANY` to limit the key further |
| `bind_external_dns_reverse_zone` / `_file` | `1.168.192.in-addr.arpa` / `192.168.1.rev` | Optional reverse zone (empty = none) |
| `bind_external_dns_update_log` | `false` | Write the post's `update_log` channel to `/var/log/named/update.log` |
| `bind_external_dns_soa_mname` / `_rname` / `_ns` | `ns1.example.com.` / `admin.example.com.` / `ns1.example.com.` | SOA and NS in the initial zone file |
| `k3s_version` | `v1.37.2+k3s1` | k3s release |

## Tested

`site.yml` was run against fresh Ubuntu 24.04 systemd containers (BIND 9.18, k3s v1.37.2+k3s1), with
`bind_external_dns_update_log: true`. The second run reported `changed=0`. `tests/verify.yml` then passed with the
NetworkPolicy active:
- `myapp.k8s.example.com` got `A <node IP>` and the TXT ownership record (TTL 0).
- The records were removed after the Service was deleted.
- `update.log` showed the `adding an RR` entries.

The RHEL code paths (`/etc/named.conf`, opening `listen-on`/`allow-query`) were not run. ansible-lint (production
profile), yamllint, `terraform validate` and tflint are clean. The Terraform code has not been applied against a real
Proxmox cluster.

## License

MIT
