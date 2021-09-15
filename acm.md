## Requirements

1. Propagation regex: `^(dev|lobdev|uat|prod)-\d+.\d+.\d+$`
1. Concurency control:
    1. Only one pipeline is allowed per environment
    1. Concurency control kicks in before approval gates

## Configuration

### parameters.yaml

1. This file replicates `subscription` CRD apart of:
    1. Parameters managed by pipeline
        1. `metadata.labels` path will be enhanced by pipeline. `acm-app` and `project` keys will be overriten by pipeline
        1. `spec.channel` path will be ignored and managed by promotion pipeline
        1. `spec.packageOverrides.packageOverrides.spec.value` path will be ignored and sourced from respective `values.yaml` file
        1. `spec.placement` path will be ignored and managed by promotion pipeline
    1. Parameters complimented by pipeline
        1. `spec.packageOverrides.packageAlias` value will be defined by pipeline from `metadatra.labels.acm-app` unless defined

#### `parameters.yaml` listing 

```yaml
```

### `promotion.yaml`

Defines promotion strategy for the application:
1. Environment order is strictly fixed and defined during repository provisioning/onboarding stage. Order example: `dev` -> `lobdev` -> `uat` -> `prod`
1. Every environment may have multiple deployment targets (`zones`) that are defined by `placementRef`s
1. Envioronment deployment targets may have priorities set for them, which allow sequential or parallel deployment of to the target zone
1. If not defined, `priority` defaults to `0`, meaning it will be executed at the first stage of application deployment to the environment

#### `promotion.yaml` listing 

```yaml
```


## Pipelines

### On pull request

Following is executed for all files in the `main` branch:
1. Basic linting
1. Resource validation:
    1. Target namespaces are allowed for deployment, ie no deployment to `kube-system`
1. Configuration validation:
    1. `placementRef`s defined in configuration are also defined under `/placement-rules`

### On merge

1. Tag code with `dev-<version>`, ie `dev-0.1.0`

### On tag

1. Request approval for execution
1. Determine target environment from tag
1. Generate ACM applications:
    1. Delete existing configuration from `/applications` folder in `release` branch
    1. Generate `application` resources based on list of apps under `/subscriptions` folder in `main` branch and store them under `/applications` folder in `release` branch
1. Generate ACM placement rules:
    1. Delete existing configuration from `/placement-rules` folder in `release` branch
    1. Copy placement rules as is into `/placement-rule` folder
1. Generate ACM subscriptions:
    1. Delete existing configuration from `/subscriptions/<env>/<zone>` folder in `release` branch based on `promotion.yaml`
    1. Generate `subscription` resources based on configuration defined under `/services/<app-name>` using value files according to `promotion.yaml` and store them under `/subscriptions/<env>/` folder
1. Commit to changes to `release` branch
1. Detect applications that changed since last release to env:
    1. Compare files with previous tag
    1. Generate list of apps based on changes to `subscription` folder
1. Wait for changes to propagate by RH ACM
1. Validate application deployment status
    1. Fail
        1. Tag current commit as `<env>-<version>-fail`
        1. Tag current commit as `<env>-<zone>-<version>-fail`
        1. Fail pipeline
    1. Success
        1. Tag current commit as `<env>-<zone>-<version>-success`
        1. Start deployment to the next zone based on priority defined in `promotion.yaml`. Go to #5
1. Promote to upper environment. This stage executed only 
    1. Tag current commit as `<env>-<version>-success`
    1. Promote to the next env by taging as `<upper_env>-<version>`



## Open issues

1. How to delete application/placements? Current problem if we delete in dev, it will also remove `applicaiton`s and `placementRef`s referenced in prod. Possible solution:
    1. Do resource addition in dev
    1. Do resource deletion and addition in prod

---

## Directory structure for `main` branch

```
```

## Directory structure for `release` branch

```
```