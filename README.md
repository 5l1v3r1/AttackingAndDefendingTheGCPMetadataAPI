# Attacking And Defending The GCPMetadata API
This repo gives an overview of some GCP metadata API attack and defend patterns

This is complementary to a presentation that I recently did with @matter_of_cat at bsidessf on this subject (will link to video when available)

## Overview
A metadata API in a cloud platform is an internal API resources like VM's that run code can query to obtain information about themselves, and obtain credentials to access the instance identity attached to the resource.

<img src="https://i.imgur.com/vW1iiFh.png" width="600">

Similar API's exist in other cloud platforms, including AWS. 

## AWS Metadata API Recap/Overview
In AWS for a long time simple get requests could be use to fetch instance identity credentials. This lead to a string of SSRF vulnerabilities, where when services would send simple get requests on behalf of a user, the user could proxy requests to the metadata API endpoint and fetch credentials.

<img src="https://i.imgur.com/6PoRmKy.png" width="400">

This is the vulnerability that lead to the [Capital One data breach](https://edition.cnn.com/2019/07/29/business/capital-one-data-breach/index.html)

### AWS Protection
AWS rolled out [IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html), which protects against most SSRF vulnerabilities, by requiring users have a header set on their request.

# GCP Metadata API
Similar to AWS, there are two versions of the metadata API, v0.1 and v1. v1 requires a header be set, which protects against many SSRF vulnerabilities.

## Bypassing the GCP Protections in Cloud Functions
Cloud functions are a serverless offering in GCP that has a metadata available to them. 

Not too long ago, a blog post was released by Google, demonstrating how to run a headless browser in these cloud functions

<img src="https://i.imgur.com/Vux88f7.png" width="400">

Because browsers can set headers, untrusted HTML can potentially access the metadata API. They are beholden to the Same Origin Policy, but [DNS rebinding](https://github.com/nccgroup/singularity) tricks [allow](https://www.youtube.com/watch?v=Q0JG_eKLcws) us to [bypass](https://www.youtube.com/watch?v=HfpnloZM61I) this restriction.

This means customers of Google that stood up headless browsers in the cloud function and allowed untrusted HTML to be rendered exposed the instance credentials to the untrusted HTML.

### Identifying Vulnerable Customers
HTTP Triggered Cloud functions expose most of their naming in their naming in their subdomain. They also default to being on the open internet with no authentication. They use the following naming convention:

```
https://<region>-<GCP-project-name>.cloudfunctions.net/<function name>
```

Where the function names default to function-1, function-2, etc... the blog post also suggested naming the function `screenshot`.

Using this information and a passiveDNS vendor, we can enumerate cloud functions:

<img src="https://i.imgur.com/CI4Pwt2.png" width="400">

At this point to get the last piece, we can guess the path via either trying `/function-X` or `/screenshot` and see if we're presented with a page that matches the one from the blog post.

Using this technique I was able to identify vulnerable customers.

_Because in GCP instances have default identities with default permissions, this leads to a compromise of a significant portion of the customer's GCP project. More on this later..._

### Fixing DNS Rebind Attacks
Google paid 1337 for the report, and added Host header validation to protect against future DNS rebind attacks. 
<img src="https://i.imgur.com/EcwsR0y.png" width="400">

All that said, SSRF is not the only way to attack the Metadata API, and the rest of this post will serve to show other methods of attack that are specific to the GCP platform.

## Google Created Identities (Service Accounts) In GCP
In GCP, Service Accounts are used to provide instances identity and give them privilege. When you enable all the API's in GCP, identities with default permissions bound to your project are created on your behalf. Some of them are attached to instances you control, some of them are attached to instances you do not control. Here's a list of them:

<img src="https://i.imgur.com/OrEPx7W.png" width="800">

These service accounts fall into two buckets, __Google Managed__ service accounts and __User Managed__ service accounts.

### User Managed Service Accounts
The main difference between these two is __User Managed__ service accounts, though created on your behalf, have permissions that are easily visible to you, and are mostly attached to resources you can see and have control over. There are two __User Managed__ service accounts, and 47 __Google Managed__ ones for comparison.

Here's what the user managed ones look like:
```
<projectID>-compute@developer.gserviceaccount.com
<projectID>@appspot.gserviceaccount.com
```

Both user managed Service Accounts have a project level Editor role binding. The editor role binding has thousands of permissions and can do things which include, access all the bigquery in your project, access all the buckets in your project, access all the databases in your project, access all the spanner in your project, etc...

<img src="https://i.imgur.com/3pWAQiU.png" width="400">

These two service accounts are attached to just about all of your resources by default. This means the cloud function example above likely meant the compromise of a lot of data and many resources, because by default the metadata API would have returned a credential that has over 2000 permissions.


### Attacking The Metadata API in GKE
In GKE, the GCP Kubernetes offering, the nodes that power your cluster are just standard GCP VM's you can view in your project. This means, again they are given the default service account with project editor.

Unlike Cloud Functions, VM's have something called _scopes_ that limit which API's the service account can access, regardless
of the service account's permissions. By default this is the scope applied to VM's:

<img src="https://i.imgur.com/g3moLzt.png" width="400">

Note: this leaves read access to storage open, meaning VM's by default can read data out of all buckets in the project.

Because workloads in GKE all run on underlying VM's that have storage open, all workloads in GKE by default can hit the metadata API, and fetch these credentials.

Note, nodes likely need storage enabled, so they can fetch from the GCR (Google Container Registry) which is powered by storage.

#### GKE Metadata Protections
GCP offeres a wide range of offerings to protect against both the K8's and the GCP metadata API:
<img src="https://i.imgur.com/yX3ZXOt.png" width="400">

You can read about them here:

+ https://cloud.google.com/kubernetes-engine/docs/how-to/protecting-cluster-metadata#concealment
+ https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity
+ https://cloud.google.com/kubernetes-engine/docs/how-to/shielded-gke-nodes


Note none are enabled by default, and the only offering that blocks the VM's GCP credentials from being fetched is Workload Identity, and it's incompatible with Metadata Concealment.

### GKE Example
Here's a demo where we enabled both Concealment and Shielded Nodes, and where able to access the node's credentials: https://drive.google.com/file/d/1JLNzBjixe_iqPSmOZfR8oE9spdnOVCp8/view

## Google Managed Service Accounts
So far we talked about service accounts created on your behalf, that by default get attached to your resources, like Cloud Functions and VM's.

But what about Google Managed Service Accounts?

For starters, it's not 100% clear how they're used, because they're mostly attached to infrastructure you can't actually see, to power the cloud. The role bindings for these service accounts are visible, which is how we took the screenshot above, but the roles themselves are often not visible, so we can't always see the underlying permissions.

Fortunately, a clever trick [@matter_of_cat](https://twitter.com/MatterOfCat) found allows us to see these permissions, via the iam roles copy API, copying the permissions from a role we can't introspect, into a role we can introspect.

<img src="https://i.imgur.com/yJwou0o.png" width="400">

Using this technique, let's look at a Google Managed Service account used for [Cloud Build](https://cloud.google.com/cloud-build)

```
<projectID>@cloudbuild.gserviceaccount.com	
```

<img src="https://i.imgur.com/LMTldtg.png" width="600">

This role has read/write to all Storage, pubsub, Cloud Build, and a few other things in your project.

### Attacking The Metadata API in Cloud Build
Cloud Build is an API that monitors a Git Repository, and builds artifacts from the repository, and typically publishes them to GCR for you.

It's pretty handy to just attach to a repo and have it magically build your container images for you.

### Cloud Build Example
Though we can't actually know exactly what's going on here, it is likely when you push to your repo, behind the scenes there's some kind of VM or Cloud Function that isn't visibile in your project doing your build. This VM as it so happens, exposes a metadata API. This means if in the build steps we reach out to the metadata API, we can steal a credential for the Google Managed service account


Note, in this demo, I've been granted 0 access to GCP, I was simply added as a collaborator on a repo, that had cloudbuild enabled. By default this is enabled on all branches, so in this demo I'll be stealing this GCP credential from a random branch on a repo I've been added as a contributor to.

Here's a demo of us doing that:
https://drive.google.com/file/d/1bISilsz1XhsNSzvvrt_WX4iLqbSe_o3y/view?usp=sharing

And here's the repository we used:
https://github.com/allisonis/fun-cloudbuild

## Detection
Fortunately, though some aspects of Google Managed service accounts are not visible in your project, logs from the usage of its credentials are.

This means we can do things like alert on anomalous behavior such as these credentials being used outside of Google IP ranges:

<img src="https://i.imgur.com/tJw6TNp.png" width="600">

Note this method is imperfect, because the attacker can likely spin up a VM in GCP and come from an IP similar to the one legitimately used.

Other anomalous behavior is API specific. For example, we can be pretty sure this cloudbuild service account shouldn't be accessing certain buckets (such as the passwords bucket used in the demo) so we can write custom alerts per API to look for anomalous behavior.

[Event Threat Detection](https://cloud.google.com/event-threat-detection/docs/concepts-overview) is an API built into Google that is meant to do this for you. It monitors stackdriver and alerts you if it sees anomalous or bad behavior. Unfortunately today it has no detection for these types of attacks, however we can expect they will continue to build this service out and hopefully pick up support for these types of attacks.

# Conclusion
Though the GCP metadata API at a high level seems similar to AWS, when we dig deeper into it we see it's the default identities GCP gives all of its offerings that really sets its risk profile apart from AWS.

Though we covered 3 API's in this readme, there are way more than 3 that expose the metadata API, some of which allow you to fetch Google Managed credentials, many of which allow you to fetch user managed credentials.

The custom header Google has you set protects against a lot of SSRF attacks, however as shown in this writeup, there are still many attacks you can perform against the metadata API that don't require an SSRF vulnerability.


## Recommendations
Here's what we recommend:

+ Across the board descope your default user managed identities
+ Don't use default identities
+ Make use of project isolation (put cloudbuild in its own project)
+ In GKE enable workload identity
+ Write custom alerts in stackdriver to detect anomalous usage of Google Managed credentials
