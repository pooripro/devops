# CD with GitHub Actions Workshop

## Preparation

* Make sure to commit your latest `bookinfo-ratings` on Google Cloud Shell to your GitHub Repository
* Make sure your `bookinfo-ratings` branch `master` and `dev` are the same
* Delete all namespaces

```bash
kubectl delete namespace student[X]-bookinfo-dev
kubectl delete namespace student[X]-bookinfo-uat
kubectl delete namespace student[X]-bookinfo-prd
```

### Practice Kubernetes

* Recreate namespace again
  * `student[X]-bookinfo-dev`
  * `student[X]-bookinfo-uat`
  * `student[X]-bookinfo-prd`
* Set default working namespace to `student[X]-bookinfo-dev`
* Create Kubernetes imagePullSecrets
* Create Kubernetes ConfigMap for MongoDB initial database script
* Deploy MongoDB with Helm to be ready for Rating Service

## Deploy Ratings Service to GCP

### Create GKE Credential Secret

* Go to <https://github.com/[GITHUB_USER]/bookinfo-ratings/settings/secrets/actions>
* Click on `New repository secret`
  * `Name:` GKE_CREDENTIALS
  * `Value:` [ASK_FROM_INSTRUCTOR]

### Improve dev-env.yml

```yaml
...
  deploy:
    needs: build
    runs-on: ubuntu-20.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: get-credentials
        uses: google-github-actions/get-gke-credentials@main
        with:
          cluster_name: k8s
          location: asia-southeast1-a
          credentials: ${{ secrets.GKE_CREDENTIALS }}
      - name: deploy
        uses: deliverybot/helm@v1
        with:
          helm: helm3
          release: bookinfo-dev-ratings
          namespace: student[X]-bookinfo-dev
          chart: k8s/helm
          value-files: k8s/helm-values/values-bookinfo-dev-ratings.yaml
```

* Check result at <http://bookinfo.dev.opsta.net/student[X]/ratings/health> and <http://bookinfo.dev.opsta.net/student[X]/ratings/ratings/1> to check the deployment

## Test and Fix CI/CD

* Try to change source code `src/ratings.js` line 207 to `Ratings is working` instead
* Push code, check GitHub Workflow and test health check page result
* Health check page won't update because when helm upgrade new release, everything still the same.

### Fix health check page won't update

* Edit `k8s/helm-values/values-bookinfo-dev-ratings.yaml` and put `COMMIT_SHA: CHANGEME` in `extraEnv:`
* Add following to deploy steps in `dev-env.yml`

```yaml
          values: |
            {
              "extraEnv": {
                "COMMIT_SHA": "${{ github.sha }}"
              }
            }
```

* Don't forget to add `COMMIT_SHA` to all helm value files in every environments
* Push code, check GitHub Workflow and test health check page result again.

## Practice GitHub Actions

### Add Acceptance Test

* Add another job to do following curl command for acceptance test to see health check page is working

```bash
curl http://bookinfo.dev.opsta.net/student[X]/ratings/health | grep "Ratings is working"
```

### Add Deploy to UAT Environment

* Create `uat-env.yml` workflow but event trigger from master branch (Don't forget to prepare ConfigMap, Secret, and deploy MongoDB in the namespace first)
* Test by pull request and merge from `dev` to `master` branch

## Assignment

* Try to use only 1 workflow file `deploy-non-prd.yml` with 1 job to follow DRY principle
* Hints

```yaml
name: deploy-non-prd
on:
  push:
    branches:
      - dev
      - master
jobs:
  build:
    runs-on: ubuntu-20.04
    steps:
      - name: Set env
        run: echo "ENV_NAME=$( [ "$GITHUB_REF" == "refs/heads/master" ] && echo 'uat' || echo ${GITHUB_REF##*/} )" >> $GITHUB_ENV
      ...
```

* Then use `${{ env.ENV_NAME }}` variable to refer to environment name
* Push, pull request to `master` branch and tag ratings service repository as `v3.0.0`

Next: [Production Deployment with GitHub Actions Workshop](13-github-actions-prd.md)
