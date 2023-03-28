# Migration

## `v1.3.26`

### Background

In this version we updated Ambassador from `1.x` to `2.x` ([See their Migration Guide](https://www.getambassador.io/docs/emissary/latest/topics/install/migrate-to-version-2/)).

Now Emissary, Ambassadors open source ingress has moved to [its own helm chart](https://github.com/emissary-ingress/emissary/tree/master/charts/emissary-ingress#introduction).
This means to two things for `byodc`.

1. The `ambassador` [chart dependency](./Chart.yaml) is now `emissary-ingress`
2. The `ambassador` config key in [values.yaml](./values.yaml) has been split up into two keys
   |                    |                                                                                                                                                       |
   |--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
   | `loadbalancer`     | Conatins the `byodc` specific configuration                                                                                                           |
   | `emissary-ingress` | Contains the `emissary-ingress` [chart configuration](https://github.com/emissary-ingress/emissary/tree/master/charts/emissary-ingress#configuration) |

### Migration Guide

#### Update `config.yaml`

1. Rename the `ambassador` key to `loadbalancer`
2. Create an `emissary-ingress` key and move the keys `install`, `replicaCount` and `scope` from the `loadbalancer` section to the new `emissary-ingress` section.

> ***Example:***
> 
> `old.config.yaml`
> ```yaml
> ambassador:
>   install: true
>   config:
>     tls:
>       enabled: false
>       crtBase64: null
>       keyBase64: null
>       crt: null
>       key: null
>     hostname: "*"
>   replicaCount: 2
>   scope:
>     singleNamespace: true
> ```
> 
> `new.config.yaml`
> ```yaml
> loadbalancer:
>   config:
>     tls:
>       enabled: false
>       crtBase64: null
>       keyBase64: null
>       crt: null
>       key: null
>     hostname: "*"
> 
> emissary-ingress:
>   install: true
>   replicaCount: 2
>   scope:
>     singleNamespace: true
> ```

#### Perform upgrade

To perform the update the old version of `byodc` needs to be removed and the old CRDs of ambassador need to be deleted.

> ***Warning:** During the update `byodc` will be completely removed and not be reachable for a few minutes.*
> 
> *If this is not acceptable a new cluster can be set up with the new `byodc` version and traffic switched to the new one after the upgrade. (See (Emissary Migraion Guide)[https://www.getambassador.io/docs/emissary/latest/topics/install/migrate-to-version-2/#install-productname-20-in-a-new-cluster])*

The following steps will update the helm repo, uninstall the helm chart, delete the CRDs and reinstall the new version of the helm chart

1. List all helm releases
   ```sh
   helm list
   ```
2. Uninstall the `byodc` release listed by the previous command.
   ```sh
   helm uninstall <release-name> -n <byodc-namespace>
   ```
   > ***Note:** If helm can not uninstall of find the release the whole namespace can be deleted and recreated like this:*
   > ```sh
   > kubectl delete namespace <byodc-namespace>`
   > `kubectl create namespace <byodc-namespace>`
   > ```
3. Delete the old CRDs
   ```sh
   kubectl delete crd -l app.kubernetes.io/name=ambassador
   ```
4. Update the helm repository
   ```sh
   helm repo update
   ```
4. Install the new helm chart
   ```sh
   helm install <release-name> fiskaltrust/bring-your-own-datacenter -f <path-to-config-yaml> -n <byodc-namespace> --version 1.3.26
   ```


> ***Example:***
> *Using release-name and namespace used in the [Quickstart Guide](https://github.com/fiskaltrust/product-de-bring-your-own-datacenter/blob/master/QuickStart.md).
>
> ```sh
> helm list
> helm uninstall bring-your-own-datacenter -n bring-your-own-datacenter
> kubectl delete crd -l app.kubernetes.io/name=ambassador
> helm repo update
> helm install bring-your-own-datcenter fiskaltrust/bring-your-own-datacenter -f new.config.yaml -n bring-your-own-datacenter --version 1.3.26
> ```

## `v1.3.45`

### Background

In this version we took the emissary ingress installation out of the helm chart.
This will fixes issues with multiple installations and will ease future updates.

The emissary ingress now needs to be installed and configured in a seperate step before the byodc helm chart is installed.

### Migration Guide

#### Update `config.yaml`

##### Emissary Ingress
The following configuration values were moved:

| old path                          | new path            |
|-----------------------------------|---------------------|
| `loadbalancer.config.tls.enabled` | `emissary.tls`      |
| `loadbalancer.config.hostname`    | `emissary.hostname` |


The `emissary-ingress` configuration section was removed.
Emissary ingress needs to be configured from [its own chart](https://artifacthub.io/packages/helm/datawire/emissary-ingress/#configuration).

##### Proxy

We've also deprecated the `byodc.proxy.http`, `byodc.proxy.https`, `byodc.proxy.no`, `byodc.proxy.ftp` and `byodc.proxy.all` variables.
These variables still work but willbe removed in a future version.

Instead you can now configure the proxy settings directly in the `byodc.proxy` variable using a [proxy string like its use in the fiskaltrust.Middleware](https://link.fiskaltrust.cloud/rollout/proxy).

#### Perform upgrade

1. List all helm releases
   ```sh
   helm list
   ```
2. Uninstall the `byodc` release listed by the previous command.
   ```sh
   helm uninstall <release-name> -n <byodc-namespace>
   ```
   > ***Note:** If helm can not uninstall of find the release the whole namespace can be deleted and recreated like this:*
   > ```sh
   > kubectl delete namespace <byodc-namespace>
   > kubectl create namespace <byodc-namespace>
   > ```
3. Install emissary ingress v3.x. Please follow the [official installation instructions](https://www.getambassador.io/docs/emissary/3.5/topics/install/helm).
   > ***Note:** When performing the `helm install` step you can also provide a `config.yaml` file to configure emissary ingress.*
4. Update the helm repository
   ```sh
   helm repo update
   ```
4. Install the new helm chart
   ```sh
   helm install <release-name> fiskaltrust/bring-your-own-datacenter -f <path-to-config-yaml> -n <byodc-namespace> --version 1.3.45-rc1 --devel
   ```
