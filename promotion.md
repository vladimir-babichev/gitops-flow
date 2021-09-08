## Scenarious

* Update module version
* Update module version and module variables
* Update module variable
* Update module variable for prod environment


## Assumptions

1. Forbid commits to `master`/`main` branch
1. Development only through pull requests
1. Pull requests always rebase from `master`/`main` before merge

## Enforcement

1. Changes only to `/dev` folder, except `*.tfvars` files

## Promotion

### Pipeline description

#### Pipeline on pull request

* Makes sure changes applied to respective env only
* Check commit message doesn't use internal format (the one used for automated commits and PRs)
* Basic linting
* Resource validation


#### Pipeline on merge to master/main
* Tag deployment version - anything not of specific format: 
  `promotion to <env>` - tagged as `dev-`
* Deploys code to respective env from head
* Validates deployment
    * if deployment
        * Success:
            * tag with `<env>-deployed`
            * create a PR to the upper env
        * Failer:
            * tag with `<env>-<version>-failed`
            * break


## Open questions:
* Against which env should I run?
* Who triggered me?
* What has changed comparing to last deployment to target env?
* How to propagate/test env specific env.tfvars