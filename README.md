# project1-gitops - what Argo CD deploys (Section 8)

GitOps repository. The code is in `project1finel`; Jenkins builds the image and writes here; Argo CD watches this repo and syncs the cluster.

```
flask-app/flask-app.yaml                       all-in-one yaml made by Jenkins (helm template) -> Argo CD Application "flask-app"
flask-aws-monitor/{dev,qa,prd}/values.yaml     per-environment values for charts/flask-app
stress-app/{dev,qa,prd}/values.yaml            per-environment values for charts/stress-app
charts/flask-app/                              copy of helm/flask-app from the code repo (Jenkins refreshes it)
charts/stress-app/                             Helm chart of the stress (load) app
bonus-apps/{nginx,redis}/                      plain manifests (bonus apps)
applicationsets/                               ApplicationSets (Git generator): flask, stress, bonus
```

## What was implemented
- **All-in-one yaml**: Jenkins runs `helm template` and commits `flask-app/flask-app.yaml`; the Argo CD Application `flask-app` (created by Terraform) syncs it into the `default` namespace.
- **ApplicationSets** (`applicationsets/`): the Git generator creates **one Argo CD Application per environment folder**:
  - `flask-aws-monitor-dev|qa|prd` - chart `charts/flask-app` + `flask-aws-monitor/<env>/values.yaml`, namespace `flask-<env>`
  - `stress-app-dev|qa|prd` - chart `charts/stress-app` + `stress-app/<env>/values.yaml`, namespace `stress-<env>`
  - `nginx`, `redis` (bonus apps) - one Application per folder in `bonus-apps/`
  Terraform creates the Argo CD Application `applicationsets` that reads this folder.
- Automated sync with `prune` and `selfHeal`: a change in Git is applied without a click.

## Environment differences (flask-aws-monitor)
| | dev | qa | prd |
|---|---|---|---|
| replicas | 1 | 2 | 3 |
| service | NodePort 30501 | NodePort 30502 | ClusterIP + Ingress (`flask-prd.example.com`) |
| cpu / memory limit | 100m / 128Mi | 200m / 256Mi | 500m / 512Mi |
| image tag | set by Jenkins on every build | promoted by hand | promoted by hand |

`stress-app`: the load grows from dev to prd; prd also stresses memory and has an HPA (see the values files).

## Missing / blockers
- The first image tag in the values files (`f6ba65b`) is an old image. After the first Jenkins build, `dev` gets the new tag by itself; `qa` and `prd` are promoted by a pull request.
- Jenkins (a bot) commits `Deploy ...` directly to `main` here. That is the GitOps hand-over; people use feature branch -> dev -> pull request.

## Deploy / update
1. `terraform apply` (repo `project1-terraform`) installs k3s + Argo CD and creates the Applications.
2. Merge to `main` in `project1finel` -> Jenkins builds -> this repo changes -> Argo CD syncs. Check: `kubectl get applications -n argocd`.
3. Change an environment: edit `flask-aws-monitor/<env>/values.yaml` (replicas, limits, service, tag) in a feature branch, merge by pull request, Argo CD applies it.
4. Promote a version: copy the tag from `dev/values.yaml` to `qa/values.yaml`, then `prd`, in a pull request.
5. Add an environment or app: add a folder `flask-aws-monitor/<new>/values.yaml` - the ApplicationSet creates the Application by itself.
6. Rollback: `git revert` the commit.

Open dev: `http://<server-ip>:30501/`, qa: `http://<server-ip>:30502/`.
