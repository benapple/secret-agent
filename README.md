# `secret-agent` - Secret generator and manager for k8s

[![PkgGoDev](https://pkg.go.dev/badge/github.com/ForgeRock/secret-agent)](https://pkg.go.dev/github.com/ForgeRock/secret-agent) [![Go Report Card](https://goreportcard.com/badge/github.com/ForgeRock/secret-agent)](https://goreportcard.com/report/github.com/ForgeRock/secret-agent) [![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](/LICENSE)

![Secret agent logo a go gopher with sunglasses and hawaiian style shirt](/assets/secretagent.png)

The `secret-agent` is a Kubernetes operator that generates the secrets required by the ForgeRock® Identity Platform. The secrets are stored in-cluster as Kubernetes secrets and can also be stored in a cloud secret manager.

## Installation

To install the latest `secret-agent` release in a Kubernetes environment, run:

```bash
kubectl apply -f https://github.com/ForgeRock/secret-agent/releases/latest/download/secret-agent.yaml
```

Specific versions of the operator can be installed by running:

```bash
SA_VERSION=v0.1.0 kubectl apply -f https://github.com/ForgeRock/secret-agent/releases/download/${SA_VERSION}/secret-agent.yaml
```

## Configuration

Once the operator is installed, new secrets can be generated by providing a secret agent configuration (SAC) object. The SAC is a custom kubernetes object monitored by the `secret-agent` operator. All the secrets’ specifications are defined through the SAC.

For more information on how to create a SAC, see the [Secret Agent Configuration Schema](#secret-agent-configuration-schema) and/or the [Examples](#examples) sections.

Once the SAC file has been created, it can be pushed to the cluster as with any other resource.

For example:

```bash
kubectl create -f config/samples/secret-agent_v1alpha1_secretagentconfiguration.yaml
```

It is important to note that the Kubernetes secrets produced by the `secret-agent` will be placed in the same namespace as the SAC. If similar secrets are desired in multiple namespaces, one SAC would be required per namespace.

## Usage

### Enabling Cloud Backup

The `secret-agent` can be configured to back up all the generated secrets in a cloud provider's secret manager solution. When this feature is enabled, secrets stored in the secret managers are considered the source of truth.

If a cloud provider has been configured, the operator will attempt to load the secret data from that cloud provider. If the secret is found in the cloud provider's secret manager, the operator will use the found data as the Kubernetes secret data. The operator will only generate new secrets if no secret data is found in the cloud provider.

The `secret-agent` supports the following cloud providers:

* Google Secret Manager
* AWS Secrets Manager
* Azure Key Vault

It is possible to run the `secret-agent` without setting up a cloud provider. This is useful when debugging or testing applications. To disable cloud provider support, set `spec.appConfig.secretsManager` to “none”.

In order to fetch and store secrets in the AWS Secrets Manager, the user must provide credentials with the necessary permissions.

#### Set up Cloud Backup With AWS Secret Manager

The `secret-agent` expects credentials to be discoverable via standard [AWS mechanisms](https://docs.aws.amazon.com/sdk-for-go/v1/developer-guide/configuring-sdk.html#specifying-credentials). These credentials can be provided in a number of ways, for example:

* Environment Variables: _AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY_
* Shared Credentials file: _~/.aws/credentials_
* Shared Configuration file: _(~/.aws/config_
* EC2 Instance Metadata (preferred): _Obtains credentials from 169.254.169.254_

Refer to [AWS documentation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access_overview.html) for instructions on how to obtain credentials and grant necessary permissions to access the AWS Secrets Manager. The `secret-agent` needs to access read/write secrets. This can be achieved by allowing access to the `arn:aws:iam::aws:policy/SecretsManagerReadWrite` AWS managed policy.

Even though the recommended way to obtain credentials is to use the EC2 Instance Metadata service, it is possible to provide custom credentials via a Kubernetes secret. The secret reference is provided in the SAC in `spec.appConfig.credentialsSecretName`. In the default `secret-agent` deployment, the user is expected to publish the cloud credentials' secret in the same namespace as the operator. This target namespace can be changed by changing the runtime argument `--cloud-secrets-namespace=[NS_NAME]` located in the operator's [manifest](/config/manager/manager.yaml). If this argument is omitted completely, the namespace will default to the namespace of each SAC.

Once these credentials are posted to a Kubernetes secret, the next step is to configure the AWS Secret Manager using the `SecretAgentConfiguration`.

For example, the following configuration targets AWS Secret Manager in `us-east-1`:

```yaml
apiVersion: secret-agent.secrets.forgerock.io/v1alpha1
kind: SecretAgentConfiguration
metadata:
  name: standard-forgerock-example
  namespace: test-sa
spec:
  appConfig:
    createKubernetesObjects: true
    credentialsSecretName: cloud-credentials [** optional**]
    secretsManager: AWS
    awsRegion: us-east-1
```

*optional for AWS*: The `cloud-credentials` secret referenced in `spec.appConfig.credentialsSecretName` would look like this:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: cloud-credentials
  namespace: test-sa
data:
  AWS_ACCESS_KEY_ID: QU....[base64 encoded key].....GY=
  AWS_SECRET_ACCESS_KEY: cRB.....[base64 encoded access key].......BB==
```

**Note: The maximum secret size supported by AWS is 65Kb** For more information, see [AWS documentation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_limits.html).

#### Set up Cloud Backup With GCP Secret Manager

The `secret-agent` expects credentials to be discoverable via standard [GCP mechanisms](https://cloud.google.com/docs/authentication). These credentials can be provided in a number of ways, including:

* [Workload identity](#workload-identity) (recommended for GKE deployments)
* [Credentials file](#credentials-file) with user accounts or service accounts.

Please refer to the [GCP Documentation](https://cloud.google.com/secret-manager/docs/reference/libraries?hl=nl#cloud-console) for instructions on how to create a service account with the necessary permissions to access the GCP Secrets Manager. The `secret-agent` needs access to read/write secrets. This can be achieved by assigning the `Secret Manager Admin` role to the service account provided.

##### Workload identity

Workload Identity is the recommended way to access Google Cloud services from applications running within GKE. For more information on how to enable workload identity see [GCP Documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#gcloud_1).

In general, once the Google service account has been created with the proper role attached and workload identity has been enabled for your GKE cluster and nodes, you can run the following commands to enable workload identity for the `secret-agent`:

```bash
PROJECTID=myproject #GCP project ID
GSA_NAME=mygcpserviceaccount #GCP service account name
# Create the GCP IAM policy binding
gcloud iam service-accounts add-iam-policy-binding --role roles/iam.workloadIdentityUser --member "serviceAccount:${PROJECTID}.svc.id.goog[secret-agent-system/secret-agent-manager-service-account]" ${GSA_NAME}@${PROJECTID}.iam.gserviceaccount.com
# Annotate the Kubernetes service account
kubectl -n secret-agent-system annotate serviceaccounts secret-agent-manager-service-account iam.gke.io/gcp-service-account=${GSA_NAME}@${PROJECTID}.iam.gserviceaccount.com
```

**Note: in order to use workload identity, no `spec.appConfig.credentialsSecretName` should be provided**. If credentials are provided, `secret-agent` will use the provided credentials instead.

##### Credentials file

The credentials are provided to the operator using a kubernetes secret under the `GOOGLE_CREDENTIALS_JSON` data key. The name of this secret is provided in `spec.appConfig.credentialsSecretName`. In the default `secret-agent` deployment, the operator will look for a secret with the provided name in the operator's own namespace. The user can specify a different namespace by setting the argument `--cloud-secrets-namespace=[NS_NAME]`. If this argument is omitted, the operator's default behavior is to fetch the credentials from the same namespace as the SAC.

The `cloud-credentials` secret referenced in `spec.appConfig.credentialsSecretName` would look like this:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: cloud-credentials
  namespace: test-sa
data:
  GOOGLE_CREDENTIALS_JSON: .....[base64 encoded service account json].....
```

##### Configure the GCP Secret Manager

Once the necessary credentials are provided to `secret-agent` using [workload identity](#workload-identity)  or a [credentials file](#credentials-file), the next step is to configure the GCP Secret Manager using the `SecretAgentConfiguration`.

For example, the following configuration targets GCP Secret Manager for the `example-project-id` project:

```yaml
apiVersion: secret-agent.secrets.forgerock.io/v1alpha1
kind: SecretAgentConfiguration
metadata:
  name: standard-forgerock-example
  namespace: test-sa
spec:
  appConfig:
    createKubernetesObjects: true
    credentialsSecretName: cloud-credentials [** skip if using workload identity **]
    secretsManager: GCP
    gcpProjectID: example-project-id
```

#### Set up Cloud Backup With Azure Key Vault

_note:_ Azure's API response time on Key Vault is long and will delay the creation of secrets. It might be beneficial to deploy a SAC before long before deploying an application if use Azure Key Vault

The `secret-agent` uses credentials which are available using two different methods: Azure Managed Identities (recommended for Azure deployemnts) or explicit credentials. Explicit credentials are configured in a secret referenced in the SAC spec `spec.appConfig.credentialsSecretName`. Example Azure Configuration for a SAC:

```yaml
apiVersion: secret-agent.secrets.forgerock.io/v1alpha1
kind: SecretAgentConfiguration
metadata:
  name: standard-forgerock-example
  namespace: test-sa
spec:
  appConfig:
    createKubernetesObjects: true
    credentialsSecretName: cloud-credentials [** optional**]
    secretsManager: Azure
    azureVaultName: secret-agent-vault
```

If no secret is provided in `credentialsSecretName`, the operator's Azure client will attempt to authenticate using managed identities. For more information, see Azure's [documentation](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity). This is the recommended configuration for deployments in Azure's AKS.

Otherwise, the credentials may be explicitly set in the `credentialsSecretName` secret. The service principle associated with the keys will need the role `Key Vault Secrets Officer` when using an RBAC policy based Key Vault.

Example credentials secret:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: cloud-credentials
data:
  # AZURE_TENANT_ID: # OPTIONAL: Update if using Azure Key Vault
  # AZURE_CLIENT_ID: # OPTIONAL: Update if using Azure Key Vault
  # AZURE_CLIENT_SECRET: # OPTIONAL: Update if using Azure Key Vault
```

### Importing your own secrets

In addition to generating secrets, the `secret-agent` allow users to import their own secrets. This is especially useful for things like certificates and certificate authorities that can be referenced by other secrets in the SAC. For example, a user can import their organization's CA and use it to sign certificates generated by the `secret-agent`.

All that is required is to provide a Kubernetes secret with the same name and same key names as described in the SAC. It is important to note that if the [cloud backup](#enabling-cloud-backup) feature is enabled, the secret to be imported must be provided using the cloud manager's secret manager. The `secret-agent` will ignore local secrets if [cloud backup](#enabling-cloud-backup) is enabled.

### Naming Convention For Cloud Backups

There is a naming convention used by `secret-agent` to store and read secrets from the cloud secret managers. In general, the names follow the format:

```bash
$prefix-$namespace-$secretName-$keyName [If secretsManagerPrefix is provided]
$namespace-$secretName-$keyName [If no prefix is provided]
```

Due to cloud provider limitations, all `/`, `.` and `_` characters in secret names are replaced by `-` when accessing the cloud secret managers.

For example, consider the following secret agent configuration:

```yaml
---
apiVersion: secret-agent.secrets.forgerock.io/v1alpha1
kind: SecretAgentConfiguration
metadata:
  name: forgerock-sac
  namespace: dev
spec:
  appConfig:
    secretsManagerPrefix: "devCluster"
    awsRegion: us-east-1
    secretsManager: AWS
  secrets:
  - name: ds-passwords
    keys:
      - name: dirmanager.pw
        type: password
```

The secret generated by the this SAC would be stored as `devCluster-dev-ds-passwords-dirmanager-pw` in the AWS Secret Manager. The same name would apply to other cloud providers.

Some key types require more than one secret to be backed up. Such key types require a public and private components stored separately. This is the case for key types: `ca`, `keypair`, `ssh` and `keytool`. These secrets use the same naming convention described above and append a suffix to the main name as constructed previously.

Key Type | Name
--- | ---
`ca`      | $NAME-pem<br>$NAME-private-pem
`keypair` | $NAME-pem<br>$NAME-private-pem
`ssh`     | $NAME<br>$NAME-pub
`keytool` | $NAME<br>$NAME-storepass<br>$NAME-keypass

In the preceding table, `$NAME` follows the convention at the top of this section.

## Examples

We provide a sample SAC that exercises all features of the `secret-agent`. See the [samples folder](config/samples).

## Secret Agent Configuration Schema

The following tables list the configurable parameters of the secret agent configuration (SAC) and their default values.

### App Config

Parameter | Description | Default
--- | --- | ---
`spec.appConfig.createKubernetesObjects` | Create Kubernetes secrets for each generated secret.  | true
`spec.appConfig.secretTimeout` | Set the timeout in seconds for generating each individual secret | 40
`spec.appConfig.secretsManager` | Select the cloud provider to target. If "none", secrets will not be backed up in any cloud secret manager.  | none
`spec.appConfig.secretsManagerPrefix` | Prefix added to the name of the secrets stored in the cloud secret manager. | ""
`spec.appConfig.credentialsSecretName` | Name of the Kubernetes secret containing the credentials to access the cloud provider. | ""
`spec.appConfig.gcpProjectID` | When using GCP as the secret mgr, specify the project ID.  | ""
`spec.appConfig.awsRegion` | When using AWS  as the secret mgr, specify the region.  | ""
`spec.appConfig.azureVaultName` | When using Azure as the secret mgr, specify the vault name. | ""
`spec.secrets` | List of Kubernetes secrets to create. See [Secret Config](#secret-config). | []

### Secret Config

Parameter | Description | Default
--- | --- | ---
`name` | Name of the Kubernetes secret to generate. | ""
`keys` | List of the specs of each key in the Kubernetes secret. See [Key Config](#key-config). | []

### Key Config

Parameter | Description | Default
--- | --- | ---
`name` | Name of the key in the Kubernetes secret. | ""
`type` | Type of key to generate. Available values: ca;literal;password;ssh;keyPair;truststore;keytool. | ""
`spec.value` | Used when key type is `literal`. Specify the value of the password. | ""
`spec.isBase64` | Used when key type is `literal`. If true, interpret the value to be used for the secret as a base64 encoded string. | false
`spec.length` | Used when key type is `password`. Specify the length of the password to generate. | 32
`spec.useBinaryCharacters` | Used when key type is `password`. If true, use the full byte range for each character, not just the ASCII range. | false
`spec.algorithm` | Used when key type is `keyPair`. Specify the algorithm used to generate the keyPair. | ""
`spec.sans` | Used when key type is `keyPair`. Specify alternate DNS names used by the certificate. | ""
`spec.selfSigned` | Used when key type is `keyPair`. If true, generate a self signed certificate. | false
`spec.signedWithPath` | Used when key type is `keyPair`. Specify the path to the CA in the SAC `secretname/keyname`. | ""
`spec.duration` | Used when key type is is `ca` or `keyPair`. Specify the valid duration of the certificate. If a negative duration is specified (-72h) the certificate is generated with an expiry date in the past. | 3650d
`spec.distinguishedName.country` | Used when key type is `ca` or `keyPair`. Specify the country name. | ""
`spec.distinguishedName.organization` | Used when key type is `ca` or `keyPair`. Specify the organization name. | ""
`spec.distinguishedName.organizationUnit` | Used when key type is `ca` or `keyPair`. Specify the organizationUnit name.  | ""
`spec.distinguishedName.locality` | Used when key type is `ca` or `keyPair`. Specify the locality name. | ""
`spec.distinguishedName.province` | Used when key type is `ca` or `keyPair`. Specify the province name. | ""
`spec.distinguishedName.streetAddress` | Used when key type is `ca` or `keyPair`. Specify the streetAddress. | ""
`spec.distinguishedName.postalCode` | Used when key type is `ca` or `keyPair`. Specify the postalCode. | ""
`spec.distinguishedName.serialNumber` | Used when key type is `ca` or `keyPair`. Specify the serialNumber. | ""
`spec.distinguishedName.commonName` | Used when key type is `ca` or `keyPair`. Specify the commonName for the certificate. | ""
`spec.truststoreImportPaths` | Used when key type is `truststore`. List of paths of certificates in the form `secretname/keyname` that will be imported into the truststore. | ""
`spec.storeType` | Used when key type is `keytool`. Specify the keystore type. Available values: pkcs12;jceks;jks. | ""
`spec.storePassPath` | Used when key type is `keytool`. Specify the path to the secret in the SAC to use as the keystore password in the form `secretname/keyname`. | ""
`spec.keyPassPath` | Used when key type is `keytool`. Specify the path to the secret in the SAC to use as the key password in the form `secretname/keyname`. | ""
`spec.keytoolAliases` | Used when key type is `keytool`. Specify the aliases to include in the keystore. See [Keytool Aliases Config](#keytool-aliases-config). | []

### Keytool Aliases Config

Parameter | Description | Default
--- | --- | ---
`name` | Name of the alias in the keystore. | ""
`cmd` | `keytool` command used to create the alias in the keystore. Supported cmds: genkeypair;genseckey;importcert;importpassword;importkeystore. | ""
`args` | Args passed to the keytool command provided in `cmd`. | ""
`sourcePath` | Used when the keystore cmd is importcert, importpassword or importkeystore. Specify the path to the secret in the SAC to import into the alias in the form `secretname/keyname`. Note: importcert only imports the public key. importkeystore must be used to import a key pair. | ""
`isKeyPair` | If importing a keypair using importkeystore, must be set to true. | false

## Runtime Arguments

Argument | Description | Default
--- | --- | ---
`--metrics-addr` | The address the metric endpoint binds to. Set to 0 to disable metrics. | ":8080"
`--health-addr` | The address the healthz/readyz endpoint binds to. | ":8081"
`--enable-leader-election` | Enable leader election for controller manager. Enabling this will ensure there is only one active controller manager. | "false"
`--cert-dir` | Directory where to store/read the webhook certs. | "/tmp/k8s-webhook-server/serving-certs"
`--cloud-secrets-namespace` | Namespace where the cloud credentials secrets are located. Defaults to the SAC namespace. | SAC's `metadata.namespace`
`--debug` | Enable debug logs. | "false"

## Running Tests

Tests can be run using `make tests`. Some of the tests exercise parts of the code that uses `keytool`, and `kubebuilder`tools. Those must be installed locally in order to run the tests. Another option is to use docker:

* Ensure you're Kubernetes context is pointing to a test cluster, such as `minikube`, then
  * `docker build -t gcr.io/forgerock-io/secret-agent-testing:latest -f --target=tester .`
  * `docker run -it --rm -v ${PWD}:/root/go/src/github.com/ForgeRock/secret-agent gcr.io/forgerock-io/secret-agent-testing:latest`
  * `go test ./...`
