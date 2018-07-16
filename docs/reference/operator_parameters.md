There are two mutually-exclusive methods to set the Postgres Operator
configuration.

* ConfigMaps-based, the legacy one.  The configuration is supplied in a
  key-value configmap, defined by the `CONFIG_MAP_NAME` environment variable.
  Non-scalar values, i.e. lists or maps, are encoded in the value strings using
  the comma-based syntax for lists and coma-separated `key:value` syntax for
  maps. String values containing ':' should be enclosed in quotes. The
  configuration is flat, parameter group names below are not reflected in the
  configuration structure. There is an
  [example](https://github.com/zalando-incubator/postgres-operator/blob/master/manifests/configmap.yaml)

* CRD-based configuration.  The configuration is stored in the custom YAML
  manifest, an instance of the custom resource definition (CRD) called
  `postgresql-operator-configuration`.  This CRD is registered by the operator
  during the start when `POSTGRES_OPERATOR_CONFIGURATION_OBJECT` variable is
  set to a non-empty value. The CRD-based configuration is a regular YAML
  document; non-scalar keys are simply represented in the usual YAML way. The
  usage of the CRD-based configuration is triggered by setting the
  `POSTGRES_OPERATOR_CONFIGURATION_OBJECT` variable, which should point to the
  `postgresql-operator-configuration` object name in the operators namespace.
  There are no default values built-in in the operator, each parameter that is
  not supplied in the configuration receives an empty value.  In order to
  create your own configuration just copy the [default
  one](https://github.com/zalando-incubator/postgres-operator/blob/wip/operator_configuration_via_crd/manifests/postgresql-operator-default-configuration.yaml)
  and change it.

CRD-based configuration is more natural and powerful then the one based on
ConfigMaps and should be used unless there is a compatibility requirement to
use an already existing configuration. Even in that case, it should be rather
straightforward to convert the configmap based configuration into the CRD-based
one and restart the operator. The ConfigMaps-based configuration will be
deprecated and subsequently removed in future releases.

Note that for the CRD-based configuration configuration groups below correspond
to the non-leaf keys in the target YAML (i.e. for the Kubernetes resources the
key is `kubernetes`). The key is mentioned alongside the group description. The
ConfigMap-based configuration is flat and does not allow non-leaf keys.

Since in the CRD-based case the operator needs to create a CRD first, which is
controlled by the `resource_check_interval` and `resource_check_timeout`
parameters, those parameters have no effect and are replaced by the
`CRD_READY_WAIT_INTERVAL` and `CRD_READY_WAIT_TIMEOUT` environment variables.
They will be deprecated and removed in the future.

Variable names are underscore-separated words.

## General

Those are top-level keys, containing both leaf keys and groups.

* **etcd_host**
  Etcd connection string for Patroni defined as `host:port`. Not required when
  Patroni native Kubernetes support is used. The default is empty (use
  Kubernetes-native DCS).

* **docker_image**
  Spilo docker image for postgres instances. For production, don't rely on the
  default image, as it might be not the most up-to-date one. Instead, build
  your own Spilo image from the [github
  repository](https://github.com/zalando/spilo).

* **sidecar_docker_images**
  a map of sidecar names to docker images for the containers to run alongside
  Spilo. In case of the name conflict with the definition in the cluster
  manifest the cluster-specific one is preferred.

* **workers**
  number of working routines the operator spawns to process requests to
  create/update/delete/sync clusters concurrently. The default is `4`.

* **max_instances**
  operator will cap the number of instances in any managed postgres cluster up
  to the value of this parameter. When `-1` is specified, no limits are applied.
  The default is `-1`.

* **min_instances**
  operator will run at least the number of instances for any given postgres
  cluster equal to the value of this parameter. When `-1` is specified, no limits
  are applied. The default is `-1`.

* **resync_period**
  period between consecutive sync requests. The default is `5m`.

## Postgres users

Parameters describing Postgres users. In a CRD-configuration, they are grouped
under the `users` key.

* **super_username**
  postgres `superuser` name to be created by `initdb`. The default is
  `postgres`.

* **replication_username**
  postgres username used for replication between instances. The default is
  `standby`.

## Kubernetes resources

Parameters to configure cluster-related Kubernetes objects created by the
operator, as well as some timeouts associated with them. In a CRD-based
configuration they are grouped under the `kubernetes` key.

* **pod_service_account_name**
  service account used by Patroni running on individual Pods to communicate
  with the operator. Required even if native Kubernetes support in Patroni is
  not used, because Patroni keeps pod labels in sync with the instance role.
  The default is `operator`.

* **pod_service_account_definition**
  The operator tries to create the pod Service Account in the namespace that
  doesn't define such an account using the YAML definition provided by this
  option. If not defined, a simple definition that contains only the name will
  be used. The default is empty.

* **pod_terminate_grace_period**
  Patroni pods are [terminated
  forcefully](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods)
  after this timeout. The default is `5m`.

* **watched_namespace**
  The operator watches for postgres objects in the given namespace. If not
  specified, the value is taken from the operator namespace. A special `*`
  value makes it watch all namespaces. The default is empty (watch the operator pod
  namespace).

* **pdb_name_format**
  defines the template for PDB (Pod Disruption Budget) names created by the
  operator. The default is `postgres-{cluster}-pdb`, where `{cluster}` is
  replaced by the cluster name. Only the `{cluster}` placeholders is allowed in
  the template.

* **secret_name_template**
  a template for the name of the database user secrets generated by the
  operator. `{username}` is replaced with name of the secret, `{cluster}` with
  the name of the cluster, `{tprkind}` with the kind of CRD (formerly known as
  TPR) and `{tprgroup}` with the group of the CRD. No other placeholders are
  allowed. The default is
  `{username}.{cluster}.credentials.{tprkind}.{tprgroup}`.

* **oauth_token_secret_name**
  a name of the secret containing the `OAuth2` token to pass to the teams API.
  The default is `postgresql-operator`.

* **infrastructure_roles_secret_name**
  name of the secret containing infrastructure roles names and passwords.

* **pod_role_label**
  name of the label assigned to the postgres pods (and services/endpoints) by
  the operator. The default is `spilo-role`.

* **cluster_labels**
  list of `name:value` pairs for additional labels assigned to the cluster
  objects. The default is `application:spilo`.

* **cluster_name_label**
  name of the label assigned to Kubernetes objects created by the operator that
  indicates which cluster a given object belongs to. The default is
  `cluster-name`.

* **node_readiness_label**
  a set of labels that a running and active node should possess to be
  considered `ready`. The operator uses values of those labels to detect the
  start of the Kubernetes cluster upgrade procedure and move master pods off
  the nodes to be decommissioned. When the set is not empty, the operator also
  assigns the `Affinity` clause to the postgres pods to be scheduled only on
  `ready` nodes. The default is empty.

* **toleration**
  a dictionary that should contain `key`, `operator`, `value` and
  `effect` keys. In that case, the operator defines a pod toleration
  according to the values of those keys. See [kubernetes
  documentation](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)
  for details on taints and tolerations. The default is empty.

* **pod_environment_configmap**
  a name of the ConfigMap with environment variables to populate on every pod.
  Right now this ConfigMap is searched in the namespace of the postgres cluster.
  All variables from that ConfigMap are injected to the pod's environment, on
  conflicts they are overridden by the environment variables generated by the
  operator. The default is empty.

## Kubernetes resource requests

This group allows you to configure resource requests for the Postgres pods.
Those parameters are grouped under the `postgres_pod_resources` key in a
CRD-based configuration.

* **default_cpu_request**
  CPU request value for the postgres containers, unless overridden by
  cluster-specific settings. The default is `100m`.

* **default_memory_request**
  memory request value for the postgres containers, unless overridden by
  cluster-specific settings. The default is `100Mi`.

* **default_cpu_limit**
  CPU limits for the postgres containers, unless overridden by cluster-specific
  settings. The default is `3`.

* **default_memory_limit**
  memory limits for the postgres containers, unless overridden by cluster-specific
  settings. The default is `1Gi`.

## Operator timeouts

This set of parameters define various timeouts related to some operator
actions, affecting pod operations and CRD creation. In the CRD-based
configuration `resource_check_interval` and `resource_check_timeout` have no
effect, and the parameters are grouped under the `timeouts` key in the
CRD-based configuration.

* **resource_check_interval**
  interval to wait between consecutive attempts to check for the presence of
  some Kubernetes resource (i.e. `StatefulSet` or `PodDisruptionBudget`). The
  default is `3s`.

* **resource_check_timeout**
  timeout when waiting for the presence of a certain Kubernetes resource (i.e.
  `StatefulSet` or `PodDisruptionBudget`) before declaring the operation
  unsuccessful. The default is `10m`.

* **pod_label_wait_timeout**
  timeout when waiting for the pod role and cluster labels. Bigger value gives
  Patroni more time to start the instance; smaller makes the operator detect
  possible issues faster. The default is `10m`.

* **pod_deletion_wait_timeout**
  timeout when waiting for the pods to be deleted when removing the cluster or
  recreating pods. The default is `10m`.

* **ready_wait_interval**
  the interval between consecutive attempts waiting for the postgres CRD to be
  created. The default is `5s`.

* **ready_wait_timeout**
  the timeout for the complete postgres CRD creation. The default is `30s`.

## Load balancer related options

Those options affect the behavior of load balancers created by the operator.
In the CRD-based configuration they are grouped under the `load_balancer` key.

* **db_hosted_zone**
  DNS zone for the cluster DNS name when the load balancer is configured for
  the cluster. Only used when combined with
  [external-dns](https://github.com/kubernetes-incubator/external-dns) and with
  the cluster that has the load balancer enabled. The default is
  `db.example.com`.

* **enable_master_load_balancer**
  toggles service type load balancer pointing to the master pod of the cluster.
  Can be overridden by individual cluster settings. The default is `true`.

* **enable_replica_load_balancer**
  toggles service type load balancer pointing to the replica pod of the
  cluster.  Can be overridden by individual cluster settings. The default is
  `false`.

* **master_dns_name_format** defines the DNS name string template for the
  master load balancer cluster.  The default is
  `{cluster}.{team}.{hostedzone}`, where `{cluster}` is replaced by the cluster
  name, `{team}` is replaced with the team name and `{hostedzone}` is replaced
  with the hosted zone (the value of the `db_hosted_zone` parameter). No other
  placeholders are allowed.

** **replica_dns_name_format** defines the DNS name string template for the
  replica load balancer cluster.  The default is
  `{cluster}-repl.{team}.{hostedzone}`, where `{cluster}` is replaced by the
  cluster name, `{team}` is replaced with the team name and `{hostedzone}` is
  replaced with the hosted zone (the value of the `db_hosted_zone` parameter).
  No other placeholders are allowed.

## AWS or GSC interaction

The options in this group configure operator interactions with non-Kubernetes
objects from AWS or Google cloud. They have no effect unless you are using
either. In the CRD-based configuration those options are grouped under the
`aws_or_gcp` key.

* **wal_s3_bucket**
  S3 bucket to use for shipping WAL segments with WAL-E. A bucket has to be
  present and accessible by Patroni managed pods. At the moment, supported
  services by Spilo are S3 and GCS. The default is empty.

* **log_s3_bucket**
  S3 bucket to use for shipping postgres daily logs. Works only with S3 on AWS.
  The bucket has to be present and accessible by Patroni managed pods. At the
  moment Spilo does not yet support this. The default is empty.

* **kube_iam_role**
  AWS IAM role to supply in the `iam.amazonaws.com/role` annotation of Patroni
  pods. Only used when combined with
  [kube2iam](https://github.com/jtblin/kube2iam) project on AWS. The default is empty.

* **aws_region**
  AWS region used to store ESB volumes. The default is `eu-central-1`.

## Debugging the operator

Options to aid debugging of the operator itself. Grouped under the `debug` key.

* **debug_logging**
  boolean parameter that toggles verbose debug logs from the operator. The
  default is `true`.

* **enable_db_access**
  boolean parameter that toggles the functionality of the operator that require
  access to the postgres database, i.e. creating databases and users. The default
  is `true`.
  
## Automatic creation of human users in the database

Options to automate creation of human users with the aid of the teams API
service. In the CRD-based configuration those are grouped under the `teams_api`
key.

* **enable_teams_api**
  boolean parameter that toggles usage of the Teams API by the operator.
  The default is `true`.

* **teams_api_url**
  contains the URL of the Teams API service. There is a [demo
  implementation](https://github.com/ikitiki/fake-teams-api). The default is
  `https://teams.example.com/api/`.

* **team_api_role_configuration**
  postgres parameters to apply to each team member role. The default is
  '*log_statement:all*'. It is possible to supply multiple options, separating
  them by commas. Options containing commas within the value are not supported,
  with the exception of the `search_path`. For instance:

  ```yaml
  teams_api_role_configuration: "log_statement:all,search_path:'data,public'"
  ```
  The default is `"log_statement:all"`

* **enable_team_superuser**
  whether to grant superuser to team members created from the Teams API.
  The default is `false`.

* **team_admin_role**
  role name to grant to team members created from the Teams API. The default is
  `admin`, that role is created by Spilo as a `NOLOGIN` role.

* **pam_role_name**
  when set, the operator will add all team member roles to this group and add a
  `pg_hba` line to authenticate members of that role via `pam`. The default is
  `zalandos`.

* **pam_configuration**
  when set, should contain a URL to use for authentication against the username
  and the token supplied as the password.  Used in conjunction with
  [pam_oauth2](https://github.com/CyberDem0n/pam-oauth2) module. The default is
  `https://info.example.com/oauth2/tokeninfo?access_token= uid
  realm=/employees`.

* **protected_roles**
  List of roles that cannot be overwritten by an application, team or
  infrastructure role. The default is `admin`.

## Logging and REST API

Parameters affecting logging and REST API listener. In the CRD-based configuration they are grouped under the `logging_rest_api` key.

* **api_port**
  REST API listener listens to this port. The default is `8080`.

* **ring_log_lines**
  number of lines in the ring buffer used to store cluster logs. The default is `100`.

* **cluster_history_entries**
  number of entries in the cluster history ring buffer. The default is `1000`.

## Scalyr options

Those parameters define the resource requests/limits and properties of the
scalyr sidecar. In the CRD-based configuration they are grouped under the
`scalyr` key.

* **scalyr_api_key**
  API key for the Scalyr sidecar. The default is empty.

* **scalyr_image**
  Docker image for the Scalyr sidecar. The default is empty.

* **scalyr_server_url**
  server URL for the Scalyr sidecar. The default is `https://upload.eu.scalyr.com`.

* **scalyr_cpu_request**
  CPU request value for the Scalyr sidecar. The default is `100m`.

* **scalyr_memory_request**
  Memory request value for the Scalyr sidecar. The default is `50Mi`.

* **scalyr_cpu_limit**
  CPU limit value for the Scalyr sidecar. The default is `1`.

* **scalyr_memory_limit**
  Memory limit value for the Scalyr sidecar. The default is `1Gi`.
