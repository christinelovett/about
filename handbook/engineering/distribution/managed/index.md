# Managed instances

This documentation details how the Distribution team at Sourcegraph internally handles the provisioning/creation/configuration/maintenance of [managed instances](https://docs.sourcegraph.com/admin/install/managed).

Please first read [the customer-facing managed instance documentation](https://docs.sourcegraph.com/admin/install/managed) to understand what these are and what we provide.

## Technical details

### Deployment type and scaling

Managed instances are Docker Compose deployments only today. We do not currently offer Kubernetes managed instances.

These managed Docker Compose deployments can scale up to the largest GCP instance type available, n1-standard-96 with 96 CPU / 360 GB memory which is typically enough for most medium to large enterprises.

We do not offer Kubernetes managed instances today as this introduces some complexity for us in terms of ongoing maintenance and overhead, we may revisit this decision in the future.

### Known limitations of managed instances

The main limitation of Docker Compose managed instances is that monthly upgrades currently involve very brief downtime (usually 1-3 minutes) and due to the single-machine deployment model an underlying GCP outage or change could result in some downtime.

### Security

- **Isolation**: Each managed instance is created in an isolated GCP project with heavy gcloud access ACLs and network ACLs for security reasons.
- **Admin access**: Both the customer and Sourcegraph personnel will have access to an application-level admin account.
- **VM/SSH access**: Only Sourcegraph personnel will have access to the actual GCP VM, this is done securely through GCP IAP TCP proxy access only. Sourcegraph personnel can make changes or provide data from the VM upon request by the customer.
- **Inbound network access**: The customer may choose between having the deployment be accessible via the public internet and protected by their SSO provider, or for additional security have the deployment restricted to an allowlist of IP addresses only (such as their corporate VPN, etc.)
- **Outbound network access**: The Sourcegraph deployment will have unfettered egress TCP/ICMP access, and customers will need to allow the Sourcegraph deployment to contact their code host. This can be done by having their code-host be publicly accessible, or by allowing the static IP of the Sourcegraph deployment to access their code host.

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

### Accessing a managed instance

Via HTTP directly to the VM (bypassing the load balancer):

```sh
gcloud compute start-iap-tunnel prod 80 --local-host-port=localhost:4444 --zone "us-central1-f" --project "sourcegraph-managed-$COMPANY"
```

SSH:

```sh
gcloud beta compute ssh --zone "us-central1-f" "prod" --tunnel-through-iap --project "sourcegraph-managed-$COMPANY"
```

### Accessing the Docker containers

Access the deployment over SSH (instructions above) and then:

```sh
sudo su
docker ps
```

You can then use regular Docker commands (e.g. `docker exec -it $CONTAINER sh`) to interact with the containers.

### Finding the external load balancer IP

```sh
$ gcloud --project=sourcegraph-managed-$COMPANY compute addresses list
NAME                 ADDRESS/RANGE  TYPE      PURPOSE  NETWORK  REGION  SUBNET  STATUS
prod-global-address  <HERE>         EXTERNAL                                    IN_USE
```

### Debugging startup scripts

View startup script logs

```
cat /var/log/syslog | grep startup-script
```

Run startup script and debug:

```
sudo google_metadata_script_runner --script-type startup --debug
```

WARNING: Running our startup script is a potentially harmful action.

More details: https://cloud.google.com/compute/docs/startupscript
