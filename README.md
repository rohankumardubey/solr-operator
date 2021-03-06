# Solr Operator
[![Latest Version](https://img.shields.io/github/tag/apache/lucene-solr-operator)](https://github.com/apache/lucene-solr-operator/releases)
[![License](https://img.shields.io/badge/LICENSE-Apache2.0-ff69b4.svg)](http://www.apache.org/licenses/LICENSE-2.0.html)
[![Go Report Card](https://goreportcard.com/badge/github.com/apache/lucene-solr-operator)](https://goreportcard.com/report/github.com/apache/lucene-solr-operator)
[![Commit since last release](https://img.shields.io/github/commits-since/apache/lucene-solr-operator/latest.svg)](https://github.com/apache/lucene-solr-operator/commits/master)
[![Docker Pulls](https://img.shields.io/docker/pulls/bloomberg/solr-operator)](https://hub.docker.com/r/bloomberg/solr-operator/)
[![Slack](https://img.shields.io/badge/slack-join_chat-white.svg?logo=slack&style=social)](https://kubernetes.slack.com/messages/solr-operator)

The __Solr Operator__ manages Apache Solr Clouds within Kubernetes. It is built on top of the [Kube Builder](https://github.com/kubernetes-sigs/kubebuilder) framework.

The project is currently in beta (`v1beta1`), and while we do not anticipate changing the API in backwards-incompatible ways there is no such guarantee yet.

If you run into issues using the Solr Operator, please:
- Reference the [version compatibility and upgrade/deprecation notes](#version-compatibility--upgrade-notes) provided below
- Create a Github Issue in this repo, describing your problem with as much detail as possible
- Reach out on our Slack channel!

Join us on the [#solr-operator](https://kubernetes.slack.com/messages/solr-operator) channel in the official Kubernetes slack workspace.

## Menu

- [Documentation](#documentation)
- [Version Compatibility and Upgrade Notes](#version-compatability--upgrade-notes)
- [Contributions](#contributions)
- [License](#license)
- [Code of Conduct](#code-of-conduct)
- [Security Vulnerability Reporting](#security-vulnerability-reporting)

## Documentation

Please visit the following pages for documentation on using and developing the Solr Operator:

- [Local Tutorial](docs/local_tutorial.md)
- [Running the Solr Operator](docs/running-the-operator.md)
- Available Solr Resources
    - [Solr Clouds](docs/solr-cloud)
    - [Solr Collections](docs/solr-collection)
    - [Solr Backups](docs/solr-backup)
    - [Solr Metrics](docs/solr-prometheus-exporter)
    - [Solr Collection Aliases](docs/solr-collection-alias)
- [Development](docs/development.md)

## Version Compatibility & Upgrade Notes

#### v0.2.7
- Do to the addition of possible sidecar/initContainers for SolrClouds, the version of CRDs used had to be upgraded to `apiextensions.k8s.io/v1`.
  
  **This means that Kubernetes support is now limited to 1.16+.**
  If you are unable to use a newer version of Kubernetes, please install the `v0.2.6` version of the Solr Operator for use with Kubernetes 1.15 and below.

- The location of backup-restore volume mounts in Solr containers has changed from `/var/solr/solr-backup-restore` to `/var/solr/data/backup-restore`.
This change was made to ensure that there were no issues using the backup API with solr 8.6+, which restricts the locations that backup data can be saved to and read from.
This change should be transparent if you are merely using the SolrBackup CRD.
All files permissions issues with SolrBackups should now be addressed.

- The default `PodManagementPolicy` for StatefulSets has been changed to `Parallel` from `OrderedReady`.
This change will not affect existing StatefulSets, as `PodManagementPolicy` cannot be updated.
In order to continue using `OrderedReady` on new SolrClouds, please use the following setting:  
`SolrCloud.spec.customSolrKubeOptions.statefulSetOptions.podManagementPolicy`

- The `SolrCloud` and `SolrPrometheusExporter` services' portNames have changed to `"solr-client"` and `"solr-metrics"` from `"ext-solr-client"` and `"ext-solr-metrics"`, respectively.
This is due to a bug in Kubernetes where `portName` and `targetPort` must match for services.

- Support for `etcd`/`zetcd` deployments has been removed.  
  The section for a Zookeeper cluster Spec `SolrCloud.spec.zookeeperRef.provided.zookeeper` has been **DEPRECATED**.
  The same fields (except for the deprecated `persistentVolumeClaimSpec` option) are now available under `SolrCloud.spec.zookeeperRef.provided`.

- Data Storage options have been expanded, and moved from their old locations.
  - `SolrCloud.spec.dataPvcSpec` has been **DEPRECATED**.  
    Please instead use the following instead: `SolrCloud.spec.dataStorage.persistent.pvcTemplate.spec=<spec>`  
  - `SolrCloud.spec.backupRestoreVolume` has been **DEPRECATED**.  
    Please instead use the following instead: `SolrCloud.spec.dataStorage.backupRestoreOptions.Volume=<volume-source>`

#### v0.2.6
- The solr-operator argument `--ingressBaseDomain` has been **DEPRECATED**.
In order to set the external baseDomain of your clouds, please begin to use `SolrCloud.spec.solrAddressability.external.domainName` instead.
You will also need to set `SolrCloud.spec.solrAddressability.external.method` to `Ingress`.
The `--ingressBaseDomain` argument is backwards compatible, and all existing SolrCloud objects will be auto-updated once your operator is upgraded to `v0.2.6`.
The argument will be removed in a future version (`v0.3.0`).

#### v0.2.4
- The default supported version of the Zookeeper Operator has been upgraded to `v0.2.6`.  
If you are using the provided zookeeper option for your SolrClouds, then you will want to upgrade your zookeeper operator version as well as the version and image of the zookeeper that you are running.
You can find examples of the zookeeper operator as well as solrClouds that use provided zookeepers in the [examples](/example) directory.  
Please refer to the [Zookeeper Operator release notes](https://github.com/pravega/zookeeper-operator/releases) before upgrading.

#### v0.2.3
- If you do not use an ingress with the Solr Operator, the Solr Hostname and Port will change when upgrading to this version. This is to fix an outstanding bug. Because of the headless service port change, you will likely see an outage for inter-node communication until all pods have been restarted.

#### v0.2.2
- `SolrCloud.spec.solrPodPolicy` has been **DEPRECATED** in favor of the `SolrCloud.spec.customSolrKubeOptions.podOptions` option.  
This option is backwards compatible, but will be removed in a future version (`v0.3.0`).

- `SolrPrometheusExporter.spec.solrPodPolicy` has been **DEPRECATED** in favor of the `SolrPrometheusExporter.spec.customKubeOptions.podOptions` option.  
This option is backwards compatible, but will be removed in a future version (`v0.3.0`).

#### v0.2.1
- The zkConnectionString used for provided zookeepers changed from using the string provided in the `ZkCluster.Status`, which used an IP, to using the service name. This will cause a rolling restart of your solrs using the provided zookeeper option, but there will be no data loss.

#### v0.2.0
- Uses `gomod` instead of `dep`
- `SolrCloud.spec.zookeeperRef.provided.zookeeper.persistentVolumeClaimSpec` has been **DEPRECATED** in favor of the `SolrCloud.zookeeperRef.provided.zookeeper.persistence` option.  
This option is backwards compatible, but will be removed in a future version (`v0.3.0`).
- An upgrade to the ZKOperator version `0.2.4` is required.

#### v0.1.1
- `SolrCloud.Spec.persistentVolumeClaim` was renamed to `SolrCloud.Spec.dataPvcSpec`

### Compatibility with Kubernetes Versions

#### Fully Compatible - v1.16+

If you require compatibility with previous versions, please install version `v0.2.6` of the Solr Operator.

## Contributions

We :heart: contributions.

Have you had a good experience with the **Solr Operator**? Why not share some love and contribute code, or just let us know about any issues you had with it?

We welcome issue reports [here](../../issues); be sure to choose the proper issue template for your issue, so that we can be sure you're providing the necessary information.

Before sending a [Pull Request](../../pulls), please make sure you read our
[Contribution Guidelines](https://github.com/bloomberg/.github/blob/master/CONTRIBUTING.md).

## License

Please read the [LICENSE](LICENSE) file here.

## Code of Conduct

This project has adopted a [Code of Conduct](https://github.com/bloomberg/.github/blob/master/CODE_OF_CONDUCT.md).
If you have any concerns about the Code, or behavior which you have experienced in the project, please
contact us at opensource@bloomberg.net.

## Security Vulnerability Reporting

If you believe you have identified a security vulnerability in this project, please send email to the project
team at opensource@bloomberg.net, detailing the suspected issue and any methods you've found to reproduce it.

Please do NOT open an issue in the GitHub repository, as we'd prefer to keep vulnerability reports private until
we've had an opportunity to review and address them.
