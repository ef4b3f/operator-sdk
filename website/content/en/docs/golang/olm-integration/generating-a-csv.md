---
title: Generating a Cluster Service Version (CSV) with Operator SDK
linkTitle: Generating CSVs
weight: 20
---

This document describes how to manage the following lifecycle for your Operator using the SDK's [`operator-sdk generate csv`][generate-csv-cli] command:

- **Generate your first release** - encapsulate the metadata needed to install your Operator and configure the permissions it needs from the generated SDK files.
- **Upgrade your Operator** - Carry over any customizations you have made and ensure a rolling update to the next version of your Operator.
- **Refresh your CRDs** - If a new version has updated CRDs, refresh those definitions within the CSV automatically.

**Note:** `operator-sdk generate csv` only officially supports Go Operators. Ansible and Helm Operators will be fully supported in the future. However, `generate csv` _may_ work with Ansible and Helm Operators if their project structure aligns with that described below.

## Configuration

### Inputs

The CSV generator requires certain inputs to construct a CSV manifest.

1. Path to the operator manifests root directory. By default `generate csv` extracts manifests from files in `deploy/` for the following kinds and adds them to the CSV. Use the `--deploy-dir` flag to change this path.
    * Roles: `role.yaml`
    * ClusterRoles: `cluster_role.yaml`
    * Deployments: `operator.yaml`
    * Custom Resources (CR's): `crds/<full group>_<version>_<kind>_cr.yaml`
    * CustomResourceDefinitions (CRD's): `crds/<full group>_<resource>_crd.yaml`
2. Path to API types root directory. The CSV generator also parses the [CSV annotations][csv-annotations] from the API type definitions to populate certain CSV fields. By default the API types directory is `pkg/apis/`. Use the `--apis-dir` flag to change this path. The CSV generator expects either of the following layouyts for the API types directory
    * Mulitple groups: `<API-root-dir>/<group>/<version>/`
    * Single groups: `<API-root-dir>/<version>/`

### Output

By default `generate csv` will generate the catalog bundle directory `olm-catalog/...` under `deploy/`. To change where the CSV bundle directory is generated use the `--ouput-dir` flag.

## Versioning

CSV's are versioned in path, file name, and in their `metadata.name` field. For example, running `operator-sdk generate csv --csv-version 0.0.1` will generate a CSV at `deploy/olm-catalog/<operator-name>/0.0.1/<operator-name>.v0.0.1.clusterserviceversion.yaml`. A versioned directory such as `deploy/olm-catalog/<operator-name>/0.0.1` is known as a [*bundle*][doc-bundle]. Versions allow the OLM to upgrade or downgrade your Operator at runtime, i.e. in a cluster. A valid semantic version is required.

`generate csv` allows you to upgrade your CSV using the `--from-version` flag. If you have an existing CSV with version `0.0.1` and want to write a new version `0.0.2`, you can run `operator-sdk generate csv --csv-version 0.0.2 --from-version 0.0.1`. This will write a new CSV manifest to `deploy/olm-catalog/<operator-name>/0.0.2/<operator-name>.v0.0.2.clusterserviceversion.yaml` containing user-defined data from `0.0.1` and any modifications you've made to `roles.yaml`, `operator.yaml`, CR's, or CRD's.

The SDK can manage CRD's in your Operator bundle as well. You can pass the `--update-crds` flag to `generate csv` to add or update your CRD's in your bundle by copying manifests in `deploy/crds` to your bundle. CRD's in a bundle are not updated by default.

## First Generation

Now that you've configured the generator, assuming version `0.0.1` is being generated, running `operator-sdk generate csv --csv-version 0.0.1` will generate a CSV defining your Operator under `deploy/olm-catalog/<operator-name>/0.0.1/<operator-name>.v0.0.1.clusterserviceversion.yaml`. No CSV existed previously in `deploy/olm-catalog/<operator-name>/0.0.1`, so no manifests were overwritten or modified.

Some fields might not have values after running `generate csv` the first time. The SDK will warn you to fill required fields and make suggestions for values for other fields:

```console
$ operator-sdk generate csv --csv-version 0.0.1
INFO[0000] Generating CSV manifest version 0.0.1
INFO[0000] Required csv fields not filled in file deploy/olm-catalog/app-operator/0.0.1/app-operator.v0.0.1.clusterserviceversion.yaml:
	spec.keywords
	spec.maintainers
	spec.provider
INFO[0000] Created deploy/olm-catalog/app-operator/0.0.1/app-operator.v0.0.1.clusterserviceversion.yaml
```

When running `generate csv` with a version that already exists, the `Required csv fields...` info statement will become a warning, as these fields are useful for displaying your Operator in Operator Hub.

**Note:** `generate csv` will populate many but not all fields in your CSV automatically. Subsequent calls to `generate csv` will warn you of missing required fields. See the list of fields [below](#csv-fields) for more information.

## Updating your CSV

Let's say you added a new CRD `deploy/crds/group.domain.com_resource_crd.yaml` to your Operator project, and added a port to your Deployment manifest `operator.yaml`. Assuming you're using the same version as above, updating your CSV is as simple as running `operator-sdk generate csv --csv-version 0.0.1`. `generate csv` will append your new CRD to `spec.customresourcedefinitions.owned`, replace the old data at `spec.install.spec.deployments` with your updated Deployment, and write an updated CSV to the same location.

The SDK will not overwrite user-defined fields like `spec.maintainers` or descriptions of CSV elements, with the exception of `specDescriptors[].displayName` and `statusDescriptors[].displayName` in `spec.customresourcedefinitions.owned`, as mentioned [above](#first-generation).

Including the `--update-crds` flag will update the CRD's in your Operator bundle.

## Upgrading your CSV

New versions of your CSV are created by running `operator-sdk generate csv --csv-version <new-version> --from-version <old-version>`. Running this command will copy user-defined fields from the old CSV to the new CSV and make updates to the new version if any manifest data was changed. This command fills the `spec.replaces` field with the old CSV versions' name.

Be sure to include the `--update-crds` flag if you want to add CRD's to your bundle alongside your CSV.

## CSV fields

Below are two lists of fields: the first is a list of all fields the SDK and OLM expect in a CSV, and the second are optional.

Several fields require user input (labeled _user_) or a [CSV annotation][csv-annotations] (labeled _annotation_). This list may change as the SDK becomes better at generating CSV's.

Required:

* `metadata.name`: a *unique* name for this CSV. Operator version should be included in the name to ensure uniqueness, ex. `app-operator.v0.1.1`.
* `spec.description` _(user)_ : a thorough description of the Operator's functionality.
* `spec.displayName` _(user)_ : a name to display for the Operator in Operator Hub.
* `spec.keywords` _(user)_ : a list of keywords describing the Operator.
* `spec.maintainers` _(user)_ : a list of human or organizational entities maintaining the Operator, with a `name` and `email`.
* `spec.provider` _(user)_ : the Operator provider, with a `name`; usually an organization.
* `spec.labels` _(user)_ : a list of `key:value` pairs to be used by Operator internals.
* `spec.version`: semantic version of the Operator, ex. `0.1.1`.
* `spec.installModes`: what mode of [installation namespacing][install-modes] OLM should use. Currently all but `MultiNamespace` are supported by SDK Operators.
* `spec.customresourcedefinitions`: any CRDs the Operator uses. Certain fields in elements of `owned` will be filled by the SDK.
    * `owned`: all CRDs the Operator deploys itself from it's bundle.
        * `name`: CRD's `metadata.name`.
        * `kind`: CRD's `metadata.spec.names.kind`.
        * `version`: CRD's `metadata.spec.version`.
        * `description` _(annotation)_ : description of the CRD.
        * `displayName` _(annotation)_ : display name of the CRD.
        * `resources` _(annotation)_ : any Kubernetes resources used by the CRD, ex. `Pod`'s and `ConfigMap`'s.
        * `specDescriptors` _(annotation)_ : UI hints for inputs and outputs of the Operator's spec.
        * `statusDescriptors` _(annotation)_ : UI hints for inputs and outputs of the Operator's status.
        * `actionDescriptors` _(user)_ : UI hints for an Operator's in-cluster actions.
    * `required` _(user)_ : all CRDs the Operator expects to be present in-cluster, if any. All `required` element fields must be populated manually.

Optional:

* `metadata.annotations.alm-examples`: CR examples, in JSON string literal format, for your CRD's. Ideally one per CRD.
* `metadata.annotations.capabilities`: level of Operator capability. See the [Operator maturity model][olm-capabilities] for a list of valid values.
* `spec.replaces`: the name of the CSV being replaced by this CSV.
* `spec.links` _(user)_ : a list of URL's to websites, documentation, etc. pertaining to the Operator or application being managed, each with a `name` and `url`.
* `spec.selector` _(user)_ : selectors by which the Operator can pair resources in a cluster.
* `spec.icon` _(user)_ : a base64-encoded icon unique to the Operator, set in a `base64data` field with a `mediatype`.
* `spec.maturity`: the Operator's maturity, ex. `alpha`.

## Further Reading

* [Information][doc-csv] on what goes in your CSV and CSV semantics.
* The original `generate csv` [design doc][doc-csv-design], which contains a thorough description how CSV's are generated by the SDK.

[doc-csv]:https://github.com/operator-framework/operator-lifecycle-manager/blob/4197455/Documentation/design/building-your-csv.md
[olm]:https://github.com/operator-framework/operator-lifecycle-manager
[generate-csv-cli]:../../cli/operator-sdk_generate_csv.md
[doc-csv-design]:https://github.com/operator-framework/operator-sdk/blob/master/doc/design/milestone-0.2.0/csv-generation.md
[doc-bundle]:https://github.com/operator-framework/operator-registry/blob/6893d19/README.md#manifest-format
[x-desc-list]:https://github.com/openshift/console/blob/70bccfe/frontend/public/components/operator-lifecycle-manager/descriptors/types.ts#L3-L35
[install-modes]:https://github.com/operator-framework/operator-lifecycle-manager/blob/4197455/Documentation/design/building-your-csv.md#operator-metadata
[olm-capabilities]:https://github.com/operator-framework/operator-sdk/blob/master/doc/images/operator-capability-level.png
[csv-annotations]: /docs/golang/olm-integration/csv-annotations/
