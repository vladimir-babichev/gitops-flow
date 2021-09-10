## Scenarious

### AS a FO-OP Engineer, I WANT TO propagate change to module version to upper environment comprised of *one region*

Regular pipeline flow - PR to Dev. Then `master` merge pipeline will take from here.

### AS a FO-OP Engineer, I WANT TO propagate change to module version to upper environment comprised of *multiple regions*

Same as above. Configuration file defining order of change deployment across regions within environment.

### Update module version and module variables

### Update module variable

* Changes added to resp

### Update module variable for prod environment


## Assumptions

1. Forbid commits to `master`/`main` branch
1. Development only through pull requests
1. Pull requests always rebase from `master`/`main` before merge
1. This is a CI pipeline for promotion of a stable code. Can't be used for development


## Enforcement

1. Changes only to `/dev` folder, except `*.tfvars` files

---

## Promotion

### Pipeline description

#### Process

1. TF cloud execution will happen in order defined in `promotion.yaml`.
1. Pipeline will orchestrate execution across `zone`s

#### Pipeline on pull request

* Makes sure changes applied to respective env only
* Check commit message doesn't use internal format (the one used for automated commits and PRs)
* Basic linting
* Resource validation


#### Pipeline on merge to master/main

* Tag deployment version - anything not of specific format: 
  `promotion to <env>` - tagged as `dev-<version>`
* Deploys code to respective env from `HEAD` (`HEAD == last tag`)
* Validates deployment
    * if deployment
        * Success:
            * tag with `<env>-<version>-success`
            * tag with `<env>-deployed`
            * create a PR to the upper env
        * Failer:
            * tag with `<env>-<version>-failed`
            * break


#### Reconcile pipeline

* Pipeline uses the same automation as promotion pipeline
* Pipeline honours `promotion.yaml`
* Pipeline checks out full repo from `HEAD` 
* Analyses tags:
    * if environment `<env>-deployed` tag != last `<env>-<version>` tag then exit
* Checkout into `<env>-deployed`
* Run Terraform
* Pipeline only applies code, doesn't modify repository or it's code - no tag settings nor commits

---

## Open questions:

* How to propagate/test env specific env.tfvars

* How to promote environment with replication configured?

---

## Improvements

### Automated promotion from dev to prod

* `terraform` folder in the root of the repository
* symlink `terraform/main.tf` code from root folder to all tiers and zones
* 
* for every merge, pipeline will treat change as a dev change and tag with `dev-<version>`
* on every tag apply code to respective env based on tag. On success promote to upper env by tagging `<upper_env>-<version>`
* Given approach doesn't imply promotion gates, due to the lack of pull requests. 
