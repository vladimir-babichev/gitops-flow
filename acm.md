## Requirements

1. Propagation regex: `^(dev|lobdev|uat|prod)-\d+.\d+.\d+$`
1. Concurency control:
    1. Only one pipeline is allowed per environment
    1. Concurency control kicks in before approval gates

## Configuration

### parameters.yaml

1. This file replicates `subscription` CRD apart of:
    1. Parameters managed by pipeline
        1. `matadata.name` is used to generate subscription name in format `name-<env>`
        1. `metadata.labels` path will be enhanced by pipeline. `acm-app` and `project` keys will be overriten by pipeline
        1. `spec.channel` path will be ignored and managed by promotion pipeline
        1. `spec.packageOverrides.packageOverrides.spec.value` path will be ignored and sourced from respective `values.yaml` file
        1. `spec.placement` path will be ignored and managed by promotion pipeline
    1. Parameters complimented by pipeline
        1. `spec.packageOverrides.packageAlias` value will be defined by pipeline from `metadatra.labels.acm-app` unless defined

#### `parameters.yaml` listing 

```yaml
metadata:
  name: namespace-config-ffdc
  namespace: ffdc
  labels:
    acm-app: namespace-config
    project: operating-platform
spec:
  channel: operating-platform-ch-helm/channel
  name: ocp-namespace-config
  packageFilter:
    version: "0.2.0"
    packageOverrides:
      - packageName: ocp-namespace-config
        packageAlias: namespace-config-ffdc
```

### `promotion.yaml`

Defines promotion strategy for the application:
1. Environment order is strictly fixed and defined during repository provisioning/onboarding stage. Order example: `dev` -> `lobdev` -> `uat` -> `prod`
1. Every environment may have multiple deployment targets (`zones`) that are defined by `placementRef`s
1. Envioronment deployment targets may have priorities set for them, which allow sequential or parallel deployment of to the target zone
1. If not defined, `priority` defaults to `0`, meaning it will be executed at the first stage of application deployment to the environment

#### `promotion.yaml` listing 

```yaml
promotion:
  dev:
    placementRef:
      - name: westeu
  lobdev:
    placementRef:
      - name: northeu
        priority: 0
      - name: westeu
  uat:
    placementRef:
      - name: northeu
        priority: 0
      - name: westeu
        priority: 1
      - name: westus
        priority: 0
      - name: eastus
        priority: 1
  prod:
    placementRef:
      - name: westus
        priority: 0
      - name: eastus
        priority: 1
```


## Pipelines

### On pull request

Following is executed for all files in the `main` branch

<img src="Images/ACM-PR-pipeline.png" alt="ACM-PR-pipeline" height="100"/>

1. Basic linting
1. Resource validation:
    1. No duplicate resource names
    1. Target namespaces are allowed for deployment, ie no deployment to `kube-system`
1. Configuration validation:
    1. `placementRef`s defined in configuration are also defined under `/placement-rules`

### On merge

<img src="Images/ACM-Merge-Pipeline.png" alt="ACM-Merge-Pipeline" height="100"/>

1. Tag code with `dev-<version>`, ie `dev-0.1.0`

### On tag

<img src="Images/ACM-Tag-Pipeline.png" alt="ACM-Tag-Pipeline" height="100"/>

1. Request approval for execution
1. Determine target environment from tag
1. Process ACM applications:
    1. Delete existing configuration from `/applications` folder in `release` branch
    1. Generate `application` resources based on list of apps under `/subscriptions` folder in `main` branch and store them under `/applications` folder in `release` branch
1. Process ACM placement rules:
    1. Delete existing configuration from `/placement-rules` folder in `release` branch
    1. Copy placement rules as is into `/placement-rule` folder
1. Process ACM subscriptions:
    1. Delete existing configuration files `/subscriptions/<env>/*-<zone>.yaml` in `release` branch based on `promotion.yaml`
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

1. How to delete application/placements? Current problem if we delete in dev, it will also remove `applicaiton`s and `placementRef`s referenced in prod. 
    1. Possible solution:
        1. Store applications and placement rules in environment specific folders, ie:
            ```
            /application/
                /dev
                    fluent-bit.yaml
                    ...
                /lobdev
                    fluent-bit.yaml
                    ...
            /placement-rules/
                /dev
                    westeu.yaml
                    ...
                /lobdev
                    northeu.yaml
                    ...
            ```

## Directory structure for `main` branch

```
.
├── placement-rules
│   ├── northeu.yaml
│   └── westeu.yaml
└── services
    ├── externaldns
    │   ├── parameters.yaml
    │   ├── promotion.yaml
    │   └── values
    │       ├── dev-northeu.yaml
    │       ├── dev-westeu.yaml
    │       ├── prod-eastus.yaml
    │       ├── prod-northeu.yaml
    │       ├── prod-westeu.yaml
    │       ├── prod-westus.yaml
    │       ├── uat-eastus.yaml
    │       ├── uat-northeu.yaml
    │       ├── uat-westeu.yaml
    │       └── uat-westus.yaml
    └── fluent-bit
        ├── parameters.yaml
        ├── promotion.yaml
        └── values
            ├── dev-northeu.yaml
            ├── dev-westeu.yaml
            ├── prod-eastus.yaml
            ├── prod-northeu.yaml
            ├── prod-westeu.yaml
            ├── prod-westus.yaml
            ├── uat-eastus.yaml
            ├── uat-northeu.yaml
            ├── uat-westeu.yaml
            └── uat-westus.yaml
```

## Directory structure for `release` branch

```
.
├── applications
│   ├── externaldns.yaml
│   └── fluent-bit.yaml
├── placement-rules
│   ├── northeu.yaml
│   └── westeu.yaml
└── subscriptions
    ├── dev
    │   ├── externaldns-westeu.yaml
    │   └── fluent-bit-westeu.yaml
    ├── lobdev
    ├── prod
    └── uat
        ├── externaldns-eastus.yaml
        ├── externaldns-northeu.yaml
        ├── externaldns-westeu.yaml
        ├── externaldns-westus.yaml
        ├── fluent-bit-eastus.yaml
        ├── fluent-bit-northeu.yaml
        ├── fluent-bit-westeu.yaml
        └── fluent-bit-westus.yaml
```