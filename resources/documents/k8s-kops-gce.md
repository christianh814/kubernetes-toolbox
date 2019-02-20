# Kops on GCE

The steps to installing/maintaining a cluster are basically the same as AWS. Here I'll outline **JUST** the differences.

## GCE Client Tools

Download and install the client tools as [outlined here](https://cloud.google.com/sdk/)

Next, initialize your client tools. This will launch a browser to log in to your Google user account. When prompted, click `Allow` to grant permission to access Google Cloud Platform resources.

```
gcloud init
```

Now, Set your default region (I used `us-west1` but you can use what you want)

```
gcloud config set compute/region us-west1
```

I set my default compute zone as well (don't worry you'll spread your load accross multiple zones with `kops`)

```
gcloud config set compute/zone us-west1-c
```

Next, verify your settings

```
gcloud config list
```

## Set up Authentication

You will need to create a service account for `kops` to use. I've highlited the steps here but it's worth it to visit the [GCE doc](https://cloud.google.com/docs/authentication/production#obtaining_and_providing_service_account_credentials_manually).

```
gcloud iam service-accounts create kops-sa
```

Grant this service account "owner" permissions

```
gcloud projects add-iam-policy-binding myproject --member "serviceAccount:kops-sa@myproject.iam.gserviceaccount.com" --role "roles/owner"
```

Download the serviceaccount config JSON

```
gcloud iam service-accounts keys create kops-sa.json --iam-account kops-sa@myproject.iam.gserviceaccount.com
```

Export `GOOGLE_APPLICATION_CREDENTIALS` to point to this file

```
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/kops-sa.json
```

## Kops Configuration

Now that you have your `gcloud` cli set up. Create a bucket to store the config

```
gsutil mb gs://kops-install/
```

Next export the bucket as `KOPS_STATE_STORE` and set `KOPS_FEATURE_FLAGS` so you can use GCE

```
export KOPS_STATE_STORE=gs://kops-install/
export KOPS_FEATURE_FLAGS=AlphaAllowGCE
```

Set the project you want to install in (this next command will populate the var with the default project you set)

```
export PROJECT=$(gcloud config get-value project)
```

Verify that you have all the env var needed for KOPS

```
$ env | egrep 'KOPS|GOOGLE|PROJECT'
GOOGLE_APPLICATION_CREDENTIALS=/path/to/kops-sa.json
KOPS_FEATURE_FLAGS=AlphaAllowGCE
KOPS_STATE_STORE=gs://kops-install/
PROJECT=myproject
```

If you're using DNS, make sure you delegated your domain to Google Cloud DNS. Verify with `dig`

```
$ dig k8s.example.com NS +short
ns-cloud-d3.googledomains.com.
ns-cloud-d2.googledomains.com.
ns-cloud-d1.googledomains.com.
ns-cloud-d4.googledomains.com.
```

## Install

Install is the same, no matter what cloud you use. The params just change. Here's an example of what I used

> **NOTE** You can use `--image "ubuntu-os-cloud/ubuntu-1604-xenial-v20170202"` to use the Ubuntu image instead of the standard COS

```
kops create cluster \
    --node-count 3 \
    --zones us-west1-a,us-west1-b,us-west1-c \
    --master-zones us-west1-a,us-west1-b,us-west1-c \
    --dns-zone k8s.example.com \
    --node-size n1-standard-2 \
    --master-size n1-standard-2 \
    --networking weave \
    --project ${PROJECT} \
    --ssh-public-key ~/.ssh/id_rsa.pub \
    --state gs://kops-install \
    --api-loadbalancer-type public \
    k8s.example.com
```

> Note you need to use `weave` or the install won't work. You **WILL** need to modify the firewalls between nodes/masters. The issue is [noted here](https://github.com/kubernetes/kops/issues/2087#issuecomment-285506042)

Then, as normal, run...

```
kops update cluster k8s.example.com --yes
```

> If you're re-installing...you can empty your bucket by running `gsutil -m rm -r gs://kops-install/*`

## Conclusion

Success! If you need more information; visit the [AWS Kops doc](k8s-kops.md) I have (most of it applies to GCE as well...the beauty of `kops`)
