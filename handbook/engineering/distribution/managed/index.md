# Managed instances

This documentation details how the Distribution team at Sourcegraph internally handles the provisioning/creation/configuration/maintenance of [managed instances](https://docs.sourcegraph.com/admin/install/managed).

Please first read [the customer-facing managed instance documentation](https://docs.sourcegraph.com/admin/install/managed) to understand what these are and what we provide.

- [Technical details](#technical-details)
  - [Deployment type and scaling](#deployment-type-and-scaling)
  - [Known limitations of managed instances](#known-limitations-of-managed-instances)
  - [Security](#security)
- [Creating a new managed instance](#creating-a-new-managed-instance)
- [Operation access](#operation-access)
  - [SSH access](#ssh-access)
  - [Port-forwarding (direct access to Caddy, Jaeger, and Grafana)](#port-forwarding-direct-access-to-caddy-jaeger-and-grafana)
  - [Access through the GCP load balancer as a user would](#access-through-the-gcp-load-balancer-as-a-user-would)
  - [Accessing the Docker containers](#accessing-the-docker-containers)
  - [Finding the external IPs](#finding-the-external-ips)
  - [Impact of recreating the instance via Terraform](#impact-of-recreating-the-instance-via-terraform)
  - []
- [Upgrade process](#upgrade-process) 
  - [1) Add a banner indicating scheduled maintenance is in progress](#1-add-a-banner-indicating-scheduled-maintenance-is-in-progress)
  - [2) Mark the database as ready-only](#2-mark-the-database-as-ready-only)
  - TODO(slimsag)
- [Startup scripts](#startup-scripts)
  - [Instance is recreated when startup script changes](#instance-is-recreated-when-startup-script-changes)
  - [Debugging startup scripts](#debugging-startup-scripts)
- [FAQ](#faq)
  - [FAQ: Why did we choose Docker Compose over Kubernetes deployments?](#faq-why-did-we-choose-docker-compose-over-kubernetes-deployments)

## Technical details

![architecture](https://storage.googleapis.com/sourcegraph-assets/managed-instance-architecture.png)

### Deployment type and scaling

Managed instances are Docker Compose deployments only today. We do not currently offer Kubernetes managed instances.

These managed Docker Compose deployments can scale up to the largest GCP instance type available, n1-standard-96 with 96 CPU / 360 GB memory which is typically enough for most medium to large enterprises.

We do not offer Kubernetes managed instances today as this introduces some complexity for us in terms of ongoing maintenance and overhead, we may revisit this decision in the future.

### Known limitations of managed instances

Sourcegraph managed instances are single-machine Docker-Compose deployments only. We do not offer Kubernetes managed instances, or multi-machine deployments, today.

With that said, Docker Compose deployments can scale up to the largest GCP instance type available, n1-standard-96 with 96 CPU & 360 GB memory, and are typically capable of supporting all but the largest of enterprises (around 25,000 repositories and 3,000 users are supported, based on what we have seen thus far.)

The main limitation of this model is that an underlying GCP infrastructure outage could result in downtime, i.e. is it not a HA deployment.

See also: [Dev FAQ: Why did we choose Docker Compose over Kubernetes deployments?](#dev-faq-why-did-we-choose-docker-compose-over-kubernetes-deployments)

### Security

- **Isolation**: Each managed instance is created in an isolated GCP project with heavy gcloud access ACLs and network ACLs for security reasons.
- **Admin access**: Both the customer and Sourcegraph personnel will have access to an application-level admin account.
- **VM/SSH access**: Only Sourcegraph personnel will have access to the actual GCP VM, this is done securely through GCP IAP TCP proxy access only. Sourcegraph personnel can make changes or provide data from the VM upon request by the customer.
- **Inbound network access**: The customer may choose between having the deployment be accessible via the public internet and protected by their SSO provider, or for additional security have the deployment restricted to an allowlist of IP addresses only (such as their corporate VPN, etc.)
- **Outbound network access**: The Sourcegraph deployment will have unfettered egress TCP/ICMP access, and customers will need to allow the Sourcegraph deployment to contact their code host. This can be done by having their code-host be publicly accessible, or by allowing the static IP of the Sourcegraph deployment to access their code host.

### Configuration management

Terraform is used to maintain all managed instances. You can find this configuration here: https://github.com/sourcegraph/deploy-sourcegraph-managed

All customer credentials, secrets, site configuration, app and user configuration - is stored in Postgres only (i.e. on the encrypted GCP disk). This allows customers to enter their access tokens, secrets, etc. directly into the app through the web UI without transferring them to us elsewhere.

## Creating a new managed instance

Creating a new managed instance involves following the steps below.

1. Ask @stephen or @beyang to create a new GCP project `sourcegraph-managed-$COMPANY` and grant you IAM **Editor** role access.
1. Ask @beyang to enable billing in the GCP project.
1. Create GCP service account credentials:
    - From console.cloud.google.com select the project > **APIs & Services** > **Credentials** > **Create credentials** > **Service account**
    - Service account name: `deploy`
    - Service account description: blank
    - On the **Service account permissions (optional)** page add the **Compute Admin**, **Storage admin**.
    - **Done** > ignore **Grant users access to this service account (optional)** and choose **Done**
    - Select **Edit** (pencil) on the service account we just created
    - **Add key** > **Create new key** > **JSON** > **Create**
1. Upload the service account key and create admin credentials in 1password:
    - Open the 1password [Managed instances vault](https://my.1password.com/vaults/l35e5xtcfsk5suuj4vfj76hqpy/allitems) (ask @stephen, @gonza, or @beyang to grant you access)
    - **Add** > **Document** > enter **$COMPANY service account** as the title > Upload the service account JSON  file previously downloaded > **Save**
    - **Add** > **Password** > enter **$COMPANY sourcegraph-admin** as the title > Change **length** to 40 and turn on symbols and digits >> **Save**
1. In GCP, enable the **Compute Engine API**:
   - Under **APIs & Services** > **Library** search for "Compute"
   - Select **Compute Engine API** and choose **Enable**
1. `export GOOGLE_APPLICATION_CREDENTIALS=~/Downloads/sourcegraph-managed-company-220df65550d4.json`
1. Clone and `cd deploy-sourcegraph-managed/`
1. `VERSION=v3.17.2 ./create-deployment.sh $COMPANY/` and **commit the result.**
1. Open and edit `deploy-sourcegraph-managed/$COMPANY/gcp-tfstate/gcp-tfstate.tf` according to the comments within, commit the result.
1. In `gcp-tfstate` run `terraform init && terraform apply && git add . && git commit -m 'initialize GCP tfstate bucket'`
1. Open and edit `infrastructure.tf` according to the comments within and commit the result.
1. In `deploy-sourcegraph-managed/$COMPANY` run `terraform init && terraform plan && terraform apply`
1. Access the instance over SSH and confirm all containers are healthy (instructions below).
1. In the infrastructure repository, [create a DNS entry](https://github.com/sourcegraph/infrastructure/blob/master/dns/sourcegraph-managed.tf) that points `$COMPANY.sourcegraph.com` to the `prod-global-address` IP following "Finding the external load balancer IP" below.
1. In the GCP web UI under **Network services** > **Load balancers** > select the load balancer > watch the SSL certificate status. It may take some time for it to become active.
1. Confirm 
1. Create a PR for review.
1. Access the instance over HTTP (instructions below)
1. Set up the initial admin account (for use by the Sourcegraph team only)
   - Email: `managed+$COMPANY@sourcegraph.com` (note `+` sign not `-`)
   - Username: `sourcegraph-admin`
   - Password: Use the password previously created and stored in 1password.
1. Configure `externalURL` in the site configuration.

## Operation access

#### SSH access

```sh
gcloud beta compute ssh --zone "us-central1-f" "prod" --tunnel-through-iap --project "sourcegraph-managed-$COMPANY"
```

#### Port-forwarding (direct access to Caddy, Jaeger, and Grafana)

This will port-forward `localhost:4444` to port `80` on the VM instance:

```sh
gcloud compute start-iap-tunnel prod 80 --local-host-port=localhost:4444 --zone "us-central1-f" --project "sourcegraph-managed-$COMPANY"
```

Replace `80` with `3370` for Grafana, or `16686` for Jaeger. Note that other ports are prevented by the `prod-allow-iap-tcp-ingress` firewall.

#### Access through the GCP load balancer as a user would

Users access Sourcegraph through GCP Load Balancer/HTTPS -> the Caddy load balancer/HTTP -> the actual sourcegraph-frontend/HTTP. If you suspect that an issue is being introduced by the GCP load balancer itself, e.g. perhaps a request is timing out there due to some misconfiguration, then you will need to access through the GCP load balancer itself. If the managed instance is protected by the load balancer firewall / not publicly accessible on the internet, then it is not possible for you to access `$COMPANY.sourcegraph.com` as a normal user would.

You can workaround this by proxying your internet traffic through the instance itself - which is allowed to reach and go through the public internet -> the GCP load balancer -> back to the instance itself. To do this, create a SOCKS5 proxy tunnel over SSH:

```ssh
bash -c '(gcloud beta compute ssh --zone "us-central1-f" --tunnel-through-iap --project "sourcegraph-managed-tinder" prod -- -N -p 22 -D localhost:5000) & sleep 600; kill $!'
```

Then test you can access it using `curl`:

```
$ curl --proxy socks5://localhost:5000 https://$COMPANY.sourcegraph.com
<a href="/sign-in?returnTo=%2F">Found</a>.
```

You can now reproduce the request using `curl`, or configure your OS or browser to use `socks5://localhost:5000`:

* Windows: Follow the “using the SOCKS proxy” section in [this article](https://www.ocf.berkeley.edu/~xuanluo/sshproxywin.html) [[mirror]](https://web.archive.org/web/20160609073255/https://www.ocf.berkeley.edu/~xuanluo/sshproxywin.html) to enable it on Internet Explorer, Edge and Firefox.
* macOS: System Preferences → Network → Advanced → Proxies → check “SOCKS proxy” and enter the host and the port.
* Linux: Most browsers have proxy settings in their Settings/Preferences.
* Command-line apps: Many CLIs accept http_proxy or https_proxy environment variables or arguments you can set the proxy. Consult the help or the manpage of the program.

**IMPORTANT**: Once you are finished, terminate the original `gcloud beta compute ssh` command so your machine's traffic is no longer going over the instance. The command above will automatically terminate after 10 minutes, to prevent this.

### Accessing the Docker containers

Access the deployment over SSH (instructions above) and then:

```sh
sudo su
docker ps
```

You can then use regular Docker commands (e.g. `docker exec -it $CONTAINER sh`) to interact with the containers.

### Finding the external IPs

```sh
$ gcloud compute addresses list --project=sourcegraph-managed-$COMPANY
NAME                  ADDRESS/RANGE  TYPE      PURPOSE  NETWORK  REGION       SUBNET  STATUS
prod-global-address   $GLOBAL_IP     EXTERNAL                                         IN_USE
prod-nat-manual-ip-0  $GLOBAL_NAT    EXTERNAL                    us-central1          IN_USE
```

- `prod-global-address` is the IP address that `$COMPANY.sourcegraph.com` should point to, it is the address of the GCP Load Balancer.
- `prod-nat-manual-ip-0` is the external IP from which egress traffic from the deployment will originate from. This is the IP the customer will need to allow to access e.g. their code host.

### Impact of recreating the instance via Terraform

All configuration about the instance itself is stored in Terraform, so recreating the instance is a non-destructive action. A brand new VM will be provisioned by Terraform, the startup script will initialize it and mount the existing data disk back into the VM, and the Sourcegraph Docker containers will start up.

This typically involves around 8m40s of downtime: 6m destroying the network endpoint group, and 2m creating the new instance / installing software / launching Docker services.

## Startup scripts

**Startup scripts are run _once_ when the instance is first created (not on each boot).**

### Instance is recreated when startup script changes

Any time a startup script is changed, Terraform will plan to delete the old VM instance and recreate it such that the script runs again.

This is a non-destructive action, aside from the fact that it involves downtime for the deployment.

### Debugging startup scripts

View startup script logs

```
cat /var/log/syslog | grep startup-script
```

Run startup script and debug:

```
sudo google_metadata_script_runner --script-type startup --debug
```

WARNING: Running our startup script twice is a potentially harmful action, as it is usually only ran once.

More details: https://cloud.google.com/compute/docs/startupscript

## Upgrade process

### 1) Add a banner indicating scheduled maintenance is in progress

TODO(slimsag) make this section public

### 2) Mark the database as ready-only

Mark the database as read-only:

```
docker exec -it pgsql psql -U sg -c 'ALTER DATABASE sg SET default_transaction_read_only = true;'
```

During this time searching will still work, but the site will refuse all database writes, e.g. this will show up in the web UI as:

> [...]: pq: cannot execute INSERT in a read-only transaction

During this time:

- Repositories will not update
- Authentication permissions will not synchronize
- LSIF precise code intel cannot be uploaded
- User settings and site configuration cannot be updated
- Extensions cannot be installed

### ...

TODO(slimsag) make this section public

## FAQ

### FAQ: Why did we choose Docker Compose over Kubernetes deployments?

Managed instances is an interim solution to having the Sourcegraph.com Cloud natively support multi-tenant private repositories. The thinking has been that:

1. Managed instances are a good interim solution because Sourcegraph.com Cloud native support for private repositories is several quarters out, at least.
2. Managed instances are something we can not invest in at all until we see customer traction, and we can tailor how much we invest based on customer traction.
3. Managed instances are something with little ongoing maintenance burden to us as long as we keep them simple, and automate as much as possible when we begin to see traction.

Because of these reasons, and due to this being an interrupt to our regularly scheduled work, a few things with Kubernetes managed instances did not make sense at our current point in time:

- We would need to set up multi-machine backup infrastructure and processes for restoration - substantially harder to do on a moments notice.
- Managing network ACLs in a Kubernetes deployment was substantially harder to do:
  - Need static NAT IPs for prospective customers to allow access to their code host which may be behind a corporate firewall.
  - Need to integrate something like GCP load balancer for SSL termination _with_ IP allow-listing (to restrict access to customer's VPN only)
- Performing upgrades in Kubernetes deployments is substantially more time consuming:
  - Being confident about addressing merge conflicts on upgrades and having them not result in unwanted changes has been difficult for us and customers.
  - We do not have proper automation in place for Kubernetes upgrades, and automating Docker Compose upgrades seemed much more straightforward.

Additionally, we noted the following:

- Docker-Compose deployments have suited companies up to 25k repos & ~3000 devs well in the past (and, frankly, beyond that..)
- We have a proper production Kubernetes deployment (sourcegraph.com) but lacked a proper Docker-Compose production deployment - this could act as that and ensure we do even more proper diligence with that deployment type and keep it functioning well for the many customers out there running it today.
- For the first customer requesting managed instances, we had a spin-up time of approx ~3d only.

None of this is to say that we will not consider switching said managed instances to Kubernetes in the future under different circumstances - it is just to say we are not doing that today.
