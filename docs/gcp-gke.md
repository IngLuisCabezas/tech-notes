# Google Cloud and GKE

> Commands used to prepare a GKE cluster and a CI deployer service account.
> Placeholders: `<ORG_ID>`, `<PROJECT_ID>`, `<SA_NAME>`, `<USER_EMAIL>`, `<CLUSTER>`.

## Organization and policy
```bash
gcloud organizations list

gcloud organizations add-iam-policy-binding <ORG_ID> \
  --member="user:<USER_EMAIL>" \
  --role="roles/orgpolicy.policyAdmin"

# Apply an org policy from a file (e.g. allow service-account key creation)
gcloud org-policies set-policy disable-key-policy.yaml
```

## Project defaults
```bash
gcloud config set project <PROJECT_ID>
gcloud config set compute/region us-central1
```

## Cluster
```bash
gcloud container clusters get-credentials <CLUSTER> --region us-central1
gcloud container clusters describe <CLUSTER> --region us-central1
gcloud container clusters delete   <CLUSTER> --region us-central1     # destructive
```

## Service account for CI (for example GitHub Actions)
```bash
gcloud iam service-accounts keys create key.json \
  --iam-account=<SA_NAME>@<PROJECT_ID>.iam.gserviceaccount.com
```

> **Secret handling**
> - `key.json` is a private credential. Store it as a CI secret (`GCP_SA_KEY`), never in git. Add `key.json` and `*.json` key files to `.gitignore`.
> - Prefer **Workload Identity Federation** over long-lived keys for GitHub Actions.
> - If a key was ever committed, **delete it in IAM** (`gcloud iam service-accounts keys delete <KEY_ID> --iam-account=...`) and create a new one. Deleting the file from git is not enough.

## Kubernetes secret for the app
```bash
kubectl create secret generic pgpassword --from-literal=PGPASSWORD="$PGPASSWORD"
```
