# s3cme

Sample Go app repo with test (on push) and release (on tag) pipelines optimized for software supply chain security (S3C). Includes Terraform setup for [OpenID Connect](https://openid.net/connect/) (IODC) in GCP with Artifact Registry, and KMS service configuration.

![](images/workflow.png)

* [Repo Usage](#usage)
* [Provenance Verification](#provenance-verification)
  * [Manual](#manual)
  * [In Cluster](#in-cluster)

What's included in the included workflow pipelines:

* PR qualification (`on-push`):
  * Source vulnerability scan using Trivy
  * Sarif-formatted report for repo alerts
* Release (`on-tag`):
  * Same test as on push
  * Image build and registry push using ko with with SBOM generation 
  * Image vulnerability scan using Trivy with max severity checks
  * Image signing using KMS key and attestation using cosign
  * SLSA provenance generation for GitHub workflow
  * SLSA provenance verification using cosign and CUE policy
* On scheule
  * Semantic code analysis using CodeQL (every 4 hours)

## Repo Usage 

1. Use this template to create a new repo (green button)

![](images/template.png)

1. Clone the new repo locally and navigate into it

```shell
git clone git@github.com:your-username/your-new-app-name.git
cd your-new-app-name
```

1. Run Terraform init

```shell
terraform -chdir=./setup init
```

1. Apply the Terraform configuration to create GCP resoruces (KMS ring/key, Artifact Registry repo, Workload Identity Pool and its Service Account)

```shell
terraform -chdir=./setup apply
```

1. When promoted, provide:

   * `project_id` - GCP project ID
   * `location` - GCP region (e.g. `us-west1`)
   * `git_repo` - The qualified name of your repo (e.g. `username/repo`)
   * `name` - Your application name (e.g. the repo portion from `git_repo`)

When completed, Terraform will output the configuration values.

1. Update `conf` job in `.github/workflows/on-tag.yaml` file to the values output by Terraform:

   * `IMG_NAME`
   * `KMS_KEY`
   * `PROVIDER_ID`
   * `REG_URI`
   * `SA_EMAIL`

1. Update Go and CUE policy file references to your own repo:

   * `./go.mod:module`
   * `./cmd/server/main.go`
   * `./policy/provenance.cue`

1. Write some code ;)

* Create pull requests (PR) to execute the test workflow (`on-push`)
* Create version tag to trigger the release workflow (`on-tag`)

## Provenance Verification  

Whenever you tag a reelase in the repo and an image is push to the registry, that image has an "attached" attestation in a form of [SLSA provenance (v0.2)](https://slsa.dev/provenance/v0.2). This allows you to trace that image all the way to its source in the repo (including the GitHub Actions that were used to generate it). That ability for vereifiable tracabiluty is called provenance. 

### Manual 

To verify the provenance of an image that was geneerated by the `on-tag` pipeline manually:

```shell
COSIGN_EXPERIMENTAL=1 cosign verify-attestation \
   --type slsaprovenance \
   --policy policy/provenance.cue \
   $IMAGE_DIGEST
```

> The `COSIGN_EXPERIMENTAL` environment variable is necessary to verify the image with the transparency log

The terminal output will include the checks that were executed as part of the validation, as well as information about subject (URI of the tag ref that triggered that workflow), with its SHA, name, and Ref.

```shell
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - Any certificates were verified against the Fulcio roots.
```

The output will also include JSON, which looks something like this (`payload` abbreviated): 

```json
{
   "payloadType": "application/vnd.in-toto+json",
   "payload": "eyJfdHl...V19fQ==",
   "signatures": [
      {
         "keyid": "",
         "sig": "MEUCIQCl+9dSv9f9wqHTF9D6L1bizNJbrZwYz0oDtjQ1wiqmLwIgE1T1LpwVd5+lOnalkYzNftTup//6H9i6wKDoCNNhpeo="
      }
   ]
}
```

The `payload` field (abbreviated) is the base64 encoded [in-toto statment](https://in-toto.io/) containing the predicate containing the GitHub Actions provenance:

```json
{
    "_type": "https://in-toto.io/Statement/v0.1",
    "predicateType": "https://slsa.dev/provenance/v0.2",
    "subject": [
        {
            "name": "us-west1-docker.pkg.dev/cloudy-s3c/s3cme/s3cme",
            "digest": {
                "sha256": "22080f8082e60e7f3ab745818e59c6f513464de23b53bbd28dc83c4936c27cbc"
            }
        }
    ],
    "predicate": {...}
}
```

### In Cluster

You can also verify the provenance of an image in your Kubernetes cluster.

> This assumes you already configured sigstore admission controller in your Kubernetes cluster. If not, you can use the provided [tools/demo-cluster](tools/demo-cluster) script to create cluster and configure sigstore policy-controller.

First, rewview the [policy/cluster.yaml](policy/cluster.yaml) file, and make sure the glob pattern matches your Artifact Registry (`**` will match any charater). You can make this as specific as you want (e.g. any image in the project in specific region)

```yaml
images:
- glob: us-west1-docker.pkg.dev/cloudy-s3c/**
```

Next, check the subject portion of the issuer identity (in this case, the SLSA workflow with the repo tag)

```yaml
identities:
- issuer: https://token.actions.githubusercontent.com
subjectRegExp: "^https://github.com/mchmarny/s3cme/.github/workflows/slsa.yaml@refs/tags/v[0-9]+.[0-9]+.[0-9]+$"
```

Finally, the policy data that checks for `predicateType` on the image should include the content of the same policy ([policy/provenance.cue](policy/provenance.cue)) we've used during the SLSA verification udirng image release and in the above manual verification process. 

```yaml
policy:
   type: cue
   data: |
     predicateType: "https://slsa.dev/provenance/v0.2"
     ...
```

> Make sure the content is indented correctly

When finished, apply the policy into the cluster:

```shell
kubectl apply -f policy/cluster.yaml
```

To verfy SLSA provnance on any namespace in your cluster, add a sigstore inclusion label to that namespace (e.g. `demo`):

```shell
kubectl label ns demo policy.sigstore.dev/include=true
```

Now, you should see an error when deploying images that don't have SLSA attestation created by your release pipeline:

```shell
kubectl run test --image=nginxdemos/hello -n demo
```

Will result in:

```shell
admission webhook "policy.sigstore.dev" denied the request
validation failed: no matching policies: spec.containers[0].image
index.docker.io/nginxdemos/hello@sha256:409564c3a1d1...
```

That policy failed because the image URI doesn't match the images glob we've specfied (`glob: us-west1-docker.pkg.dev/cloudy-s3c/s3cme/**`). How about if we try to deploy image that does:

```shell
kubectl run test -n demo --image us-west1-docker.pkg.dev/cloudy-s3c/s3cme/s3cme@sha256:ead1a5b3e83760e304ee449732feaa4a80c76c2c098dccb56e3d3b8926b5509d
```

Now the failure is on the SLSA plicy due to lack of verifiable attestations:

```shell
admission webhook "policy.sigstore.dev" denied the request:
validation failed: failed policy: slsa-attestation-image-policy: spec.containers[0].image
us-west1-docker.pkg.dev/cloudy-s3c/s3cme/s3cme@sha256:ead1a5b3e83760e304ee449732feaa4a80c76c2c098dccb56e3d3b8926b5509d 
attestation keyless validation failed for authority authority-0 for us-west1-docker.pkg.dev/cloudy-s3c/s3cme/s3cme@sha256:ead1a5b3e83760e304ee449732feaa4a80c76c2c098dccb56e3d3b8926b5509d: 
no matching attestations:
```

## Disclaimer

This is my personal project and it does not represent my employer. While I do my best to ensure that everything works, I take no responsibility for issues caused by this code.
