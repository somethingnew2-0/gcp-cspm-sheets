Simple CSPM
====

This project runs a Google App Script inside of Google Sheets to daily collect useful audit
data from several sources in Google Cloud Platform (GCP) for Cloud Security Posture Management.

Google Sheets is used for maximum customizability and minimum operational maintenance requirements
using "serverless" Google App Scripts.

The following sources in GCP are used to collect data:
* Cloud Asset Inventory
    * Search All Assets
    * Search IAM Policies
* Recommenders
* Insights
* Cloud Security Command Center
    * Findings
* API Keys API

I'm [Peter C (@itspeterc)](https://twitter.com/itspeterc), feel free star this repository and follow on Twitter for more cloud security insights!

Shout out to [Matthew Bryant (@IAmMandatory)](https://twitter.com/IAmMandatory) and his DEF CON 29 talk on
[Hacking G Suite: The Power of Dark Apps Script Magic](https://www.youtube.com/watch?v=6AsVUS79gLw) for inspiring this project.

## Installation
1. Make a copy of this Google Sheet by clicking "File" -> "Make a Copy"
2. Update your GCP project to run from
3. Add the following GCP IAM roles for your user on your GCP project to run from
    * roles/serviceusage.serviceUsageAdmin
4. [Enable "Service Usage API"](https://console.cloud.google.com/apis/api/serviceusage.googleapis.com/overview) on your GCP Project to run from
5. Add the following GCP IAM roles for your user on your GCP organization
    * *roles/cloudasset.viewer*
    * *roles/recommender.iamViewer*
    * *roles/recommender.projectUtilViewer*
    * *roles/recommender.cloudAssetInsightsViewer*
    * *roles/recommender.firewallViewer*
    * *roles/serviceusage.apiKeysViewer*
    * *roles/securitycenter.findingsViewer*
6. Click "Run Audit"
7. Approve Google Sheets Permissions to Run
8. Click "Run Audit" Again

## Customize
After making your own copy of the Google Sheet, click "Tools" -> "Script editor" to modify
the javascript App Script code also included in this repository as [Code.gs](Code.gs).

## Audit Data Collected in Sheets
### Public Assets from Cloud Asset Inventory

An unfortunately common vulnerability with cloud deployments is unintentionally publicly accessible resources. The first several sheets described below use the [Cloud Asset Inventory service](https://cloud.google.com/asset-inventory/docs/overview)
to list public assets as well describe how to use [Organization Policies](https://cloud.google.com/resource-manager/docs/organization-policy/overview)
to restrict future public exposure. See the [Limiting public IPs on Google Cloud blog post and video](https://cloud.google.com/blog/topics/developers-practitioners/limiting-public-ips-google-cloud)
for an excellent summary on using organization policies to restrict public IPs for most Google Cloud resources.

#### Public GCE VMs

This sheet contains the list of running Compute Engine instances with external (aka. public)
IP addresses attached. This can be useful for enabling the
[`compute.vmExternalIpAccess` organization policy](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address#disableexternalip) when creating an initial list of projects to be exempted from the policy.

While removing external IP addresses in favor of accessing Compute Engine instances via internal
IP addresses using [Google External Load Balancers](https://cloud.google.com/load-balancing/docs/network)
or the [Identity Aware Proxy](https://cloud.google.com/iap/docs/using-tcp-forwarding),
[VPC firewall rules](https://cloud.google.com/vpc/docs/firewalls) can also be used to limit
the destination ports, destination protocols, and source IP addresses that can access the instances
as an alternative.

Below is an equivalent [Cloud Asset Inventory `gcloud` command](https://cloud.google.com/asset-inventory/docs/listing-assets) used to generate a CSV of this sheet for your organization by specifying an `$ORGANIZATION_ID`.
```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='compute.googleapis.com/Instance' \
  --filter="resource.data.networkInterfaces[].accessConfigs[].name='External NAT' AND resource.data.status='RUNNING'" \
  --format="csv(name.scope(projects).segment(0), resource.data.name, resource.data.networkInterfaces[].accessConfigs[0].natIP, resource.data.status, resource.data.creationTimestamp, resource.data.lastStartTimestamp)" > public_instances.csv
```

#### Public CloudSQL Instances

This sheet contains the list of running CloudSQL database instances with a [public ip address](https://cloud.google.com/sql/docs/mysql/configure-ip).  This can be useful for enabling the
[`sql.restrictPublicIp` organization policy](https://cloud.google.com/sql/docs/mysql/connection-org-policy#connection-constraints) when creating an initial list of projects to be exempted from the policy.

Keep in mind that CloudSQL database instances can have [authorized networks](https://cloud.google.com/sql/docs/mysql/authorize-networks)
which limit the sources from where the database with a public instance can be accessed
on the internet, although [private IP connectivity](https://cloud.google.com/sql/docs/mysql/private-ip)
should be preferred for security along with [Cloud SQL Auth proxy](https://cloud.google.com/sql/docs/mysql/connect-admin-proxy) or [IAM database authentication](https://cloud.google.com/sql/docs/mysql/authentication).

Below is an equivalent [Cloud Asset Inventory `gcloud` command](https://cloud.google.com/asset-inventory/docs/listing-assets) used to generate a CSV of this sheet for your organization by specifying an `$ORGANIZATION_ID`.
 ```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='sqladmin.googleapis.com/Instance' \
  --filter="resource.data.settings.activationPolicy='ALWAYS' AND resource.data.settings.ipConfiguration.ipv4Enabled='TRUE'" \
  --format="csv(resource.data.project, resource.data.name, resource.data.gceZone, resource.data.settings.ipConfiguration.ipv4Enabled, resource.data.settings.ipConfiguration.requireSsl, resource.data.serverCaCert.createTime, resource.data.settings.activationPolicy)" > public_cloudsql_instances.csv
```

#### Public Cloud Functions

This sheet contains the list of running Cloud Functions with an [HTTPS Trigger](https://cloud.google.com/functions/docs/calling/http) and
[ingress settings](https://cloud.google.com/functions/docs/networking/network-settings#ingress_settings)
allowing all traffic as opposed to restricting it to internal VPC traffic or Cloud Load Balancing.
By default, Cloud Functions require IAM authentication and to make a function truly public it needs to
have an [unauthenticated invocation IAM binding](https://cloud.google.com/functions/docs/securing/managing-access-iam#allowing_unauthenticated_http_function_invocation) set after January 15, 2020
which this sheet does also check. The results in this sheet will overlap with the [Public IAM Policies sheet below](#public-iam-policies) for
Cloud Functions which have either `allUsers` or `allAuthentication` invocation IAM bindings. Because of this both the
[`iam.allowedPolicyMemberDomains`](https://cloud.google.com/resource-manager/docs/organization-policy/restricting-domains) (Recommended)
and the [`cloudfunctions.allowedIngressSettings`](https://cloud.google.com/resource-manager/docs/organization-policy/org-policy-constraints)
organization policies can be used to restrict Cloud Functions from being made public.

Below are two similar [Cloud Asset Inventory `gcloud` commands](https://cloud.google.com/asset-inventory/docs/listing-assets) used to generate a CSV of this sheet for your organization by specifying an `$ORGANIZATION_ID`.
```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='cloudfunctions.googleapis.com/CloudFunction' \
  --filter="resource.data.status='ACTIVE' AND resource.data.list(show="keys"):'httpsTrigger' AND resource.data.ingressSettings='ALLOW_ALL'" \
  --format="csv(resource.data.httpsTrigger.url)" > public_cloud_functions.csv
```
```
gcloud beta asset search-all-iam-policies --scope="organizations/$ORGANIZATION_ID" --query='memberTypes:("allUsers" OR "allAuthenticatedUsers") AND policy.role.permissions:cloudfunctions.functions.invoke'
```

#### Public GKE Clusters

```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='container.googleapis.com/Cluster' \
  --filter="resource.data.privateClusterConfig.enabledPrivateEndpoint AND resource.data.status='RUNNING'" \
  --format="csv(name.scope(projects).segment(0), resource.data.name, resource.data.endpoint, resource.data.privateClusterConfig.enablePrivateEndpoint, resource.data.masterAuthorizedNetworksConfig.cidrBlocks, resource.data.status, resource.data.createTime)" > public_clusters.csv
```

#### Public App Engine

```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='appengine.googleapis.com/Service'
```
```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='appengine.googleapis.com/Application'
```

#### Public Cloud Run

```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='run.googleapis.com/Service'
```

#### External Load Balancers from Cloud Asset Inventory

##### External Global Forwarding Rules

```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='compute.googleapis.com/GlobalForwardingRule' \
  --filter="resource.data.loadBalancingScheme='EXTERNAL'" \
  --format="csv(name.scope(projects).segment(0), resource.data.name, resource.data.IPAddress, resource.data.portRange, resource.data.loadBalancingScheme, resource.data.creationTimestamp)" > external_global_forwarding_rule.csv
```

##### External Forwarding Rules

```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='compute.googleapis.com/ForwardingRule' \
  --filter="resource.data.loadBalancingScheme='EXTERNAL'" \
  --format="csv(name.scope(projects).segment(0), resource.data.name, resource.data.IPAddress, resource.data.portRange, resource.data.loadBalancingScheme, resource.data.creationTimestamp)" > external_forwarding_rule.csv
```

##### External Backend Services

```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='compute.googleapis.com/BackendService' \
  --filter="resource.data.loadBalancingScheme='EXTERNAL'" \
  --format="csv(name.scope(projects).segment(0), resource.data.name, resource.data.protocol, resource.data.port, resource.data.loadBalancingScheme, resource.data.creationTimestamp)" > external_backend_service.csv
```

##### External Regional Backend Services

```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='compute.googleapis.com/RegionBackendService' \
  --filter="resource.data.loadBalancingScheme='EXTERNAL'" \
  --format="csv(name.scope(projects).segment(0), resource.data.name, resource.data.protocol, resource.data.port, resource.data.loadBalancingScheme, resource.data.creationTimestamp)" > external_regional_backend_service.csv
```

### Public IAM Policies


```
gcloud beta asset search-all-iam-policies --scope="organizations/$ORGANIZATION_ID" \
  --query='memberTypes:("allUsers" OR "allAuthenticatedUsers")'
```

### Recommenders

#### Unused Projects

```
gcloud projects list --format="value(projectId)" | xargs -t -I {} \
  gcloud recommender recommendations list --project={} --billing-project=$OPERATING_PROJECT \
    --recommender=google.resourcemanager.projectUtilization.Recommender \
    --filter="recommenderSubtype=CLEANUP_PROJECT" --location=global
```

#### IAM Recommendations

```
gcloud projects list --format="value(projectId)" | xargs -t -I {} \
  gcloud recommender recommendations list --project={} --billing-project=$OPERATING_PROJECT \
    --recommender=google.iam.policy.Recommender \
    --filter="stateInfo.state=ACTIVE" --location=global
```

### Insights

#### Lateral Movement Insights
```
gcloud projects list --format="value(projectId)" | xargs -t -I {} \
  gcloud recommender insights list --project={} --billing-project=$OPERATING_PROJECT \
    --insight-type=google.iam.policy.LateralMovementInsight \
    --filter="stateInfo.state=ACTIVE" --location=global
```

#### IAM Policy Insights
```
gcloud recommender insights list --organization=$ORGANIZATION_ID --billing-project=$OPERATING_PROJECT \
  --insight-type=google.iam.policy.Insight \
  --filter="stateInfo.state=ACTIVE" --location=global
```
```
gcloud beta asset list --organization=$ORGANIZATION_ID --content-type='resource' \
  --asset-types='cloudresourcemanager.googleapis.com/Folder' \
  --format="value(resource.data.name.segment(1))" | xargs -t -I {} \
    gcloud recommender insights list --folder={} --billing-project=$OPERATING_PROJECT \
      --insight-type=google.iam.policy.Insight \
      --filter="stateInfo.state=ACTIVE" --location=global
```
```
gcloud projects list --format="value(projectId)" | xargs -t -I {} \
  gcloud recommender insights list --project={} --billing-project=$OPERATING_PROJECT \
    --insight-type=google.iam.policy.Insight \
    --filter="stateInfo.state=ACTIVE" --location=global
```

#### Service Account Insights
```
gcloud projects list --format="value(projectId)" | xargs -t -I {} \
  gcloud recommender insights list --project={} --billing-project=$OPERATING_PROJECT \
    --insight-type=google.iam.serviceAccount.Insight \
    --filter="stateInfo.state=ACTIVE" --location=global
```

#### Asset Insights
```
gcloud projects list --format="value(projectId)" | xargs -t -I {} \
  gcloud recommender insights list --project={} --billing-project=$OPERATING_PROJECT \
    --insight-type=google.cloudasset.asset.Insight \
    --filter="stateInfo.state=ACTIVE" --location=global
```

#### Firewall Insights
```
gcloud projects list --format="value(projectId)" | xargs -t -I {} \
  gcloud recommender insights list --project={} --billing-project=$OPERATING_PROJECT \
    --insight-type=google.compute.firewall.Insight \
    --filter="stateInfo.state=ACTIVE" --location=global
```


### API Keys
```
gcloud projects list --format="value(projectId)" | xargs -t -I {} \
  gcloud alpha services api-keys list --project={} --billing-project=$OPERATING_PROJECT \
    --format="csv(name.segement(1), displayName, uid, createTime)"
```

## Audit Data not yet Collected
* API Keys
* Cloud Security Command Center (CSCC) Findings
* VM Manager Vulnerabilities

## Other Free and Open-Source Alternatives
* [OpenCSPM](https://github.com/OpenCSPM/opencspm)
* [Forseti](https://forsetisecurity.org)
* [Cloud Custodian](https://cloudcustodian.io)
