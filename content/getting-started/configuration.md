---
title: Other configuration
description: Configure Estafette to tune it to your needs
weight: 20
---

## Config.yaml

While _Estafette_'s goal is to keep as much control as possible with the individual application in the `.estafette.yaml` manifest, unavoidably there's some configuration that needs to be done centrally. This configuration is stored in the `config.yaml` file. Below is a description of the various configuration sections.

### Integrations

For integrations with 3rd party services the `integrations` section provides a place for configuration for each individual service.

#### Github

See [Github integration]({{< relref "github-integration" >}}).

#### Bitbucket

See [Bitbucket integration]({{< relref "bitbucket-integration" >}}).

#### Slack

The Slack integration allows a Slash command to be configured in order to provide functionalities like encrypting a secret or triggering a release directly from Slack. For the Slash command to integrate with _Estafette_ the following configuration section is needed:

```yaml
integrations:
  slack:
    enable: true
    clientID: '<client id>'
    clientSecret: estafette.secret(***)
    appVerificationToken: estafette.secret(***)
    appOAuthAccessToken: estafette.secret(***)
```

#### Pub/Sub

In order for the Estafette CI api to create subscriptions to topics used in _pubsub_ triggers and validate incoming events the following config section needs to be added:

```yaml
integrations:
  ...
  pubsub:
    enable: true
    defaultProject: '<project id where estafette-ci-api runs>'
    endpoint: 'https://<public hostname for integrations>/api/integrations/pubsub/events'
    audience: somerandomaudiencekey
    serviceAccountEmail: '<email address for service account used by estafette-ci-api>'
    subscriptionNameSuffix: ~estafette-ci-pubsub-trigger
    subscriptionIdleExpirationDays: 365
```

For all projects used in _pubsub_ triggers the _estafette-ci-api_ service account needs the following roles:

```yaml
- pubsub.subscriber
- pubsub.editor
```

### API Server

In order to set correct links for build status integration and provide correct communication between the builder jobs and the api there's some minimal configuration for the API itself:

```yaml
apiServer:
  baseURL: 'https://<(private) host for the web gui>'
  integrationsURL: 'https://<public host to receive webhooks>'
  serviceURL: 'http://estafette-ci-api.<namespace>.svc.cluster.local/'
```

With the helm chart this config is set from the following Helm values:

```yaml
# (private) host at which to access the web interface and api
baseHost: estafette.mydomain.com

# host to receive webhooks from 3rd party integrations like github, bitbucket
integrationsHost: estafette-integrations.mydomain.com
```

You'll only need to set it when passing a custom config file.

### Authentication

Currently the only supported user authentication is by using Google's Identity Aware Proxy (IAP); all authenticated users are allowed to use all functions on all repositories until role-based access (RBAC) is implemented. _Estafette_ can operate without IAP, but any manual actions will have to be triggered via Slack integration for now.

The API key is used to secure communication from the builder jobs to the API.

```yaml
auth:
  iap:
    enable: true
    audience: '<identity aware proxy>'
  apiKey: estafette.secret(***)
```

### Database

_Estafette_ supports Postgresql; this allows it to use CockroachDB as it's database. Below is the required configuration to connect to the database.

```yaml
database:
  databaseName: defaultdb
  host: estafette-ci-db-public
  insecure: false
  sslMode: verify-full
  certificateAuthorityPath: /cockroach-certs/ca.crt
  certificatePath: /cockroach-certs/tls.crt
  certificateKeyPath: /cockroach-certs/tls.key
  port: 26257
  user: root
  password: ''
```

### Credentials

In order to centrally manage credentials used by various _Estafette_ extensions or your own custom extensions. The only required fields are `name` and `type`, other fields are fully up to the consumer of the credentials. This allows _Estafette_ to be easily extended with new types of credentials used by _trusted images_ (see section below).

Types supported by _Estafette_'s various official extensions are `container-registry`, `kubernetes-engine` and `slack-webhook` amongst others.

```yaml
credentials:
- name: 'container-registry-estafette'
  type: 'container-registry'
  repository: 'estafette'
  private: false
  username: 'estafette.secret(793-0QLfe5QN3n1k.mDk5fL-_urybZ3T9Z3famkSZR68d-SrfqA==)'
  password: 'estafette.secret(3p0Geq5U12j98HXV.Eaupah0YHK6nQkMgT4WYzC5R8FRQbDk5H6aTo1saw35de2KQ)'
- name: 'container-registry-extensions'
  type: 'container-registry'
  repository: 'extensions'
  private: false
  username: 'estafette.secret(793-0QLfe5QN3n1k.mDk5fL-_urybZ3T9Z3famkSZR68d-SrfqA==)'
  password: 'estafette.secret(3p0Geq5U12j98HXV.Eaupah0YHK6nQkMgT4WYzC5R8FRQbDk5H6aTo1saw35de2KQ)'
- name: 'gke-production'
  type: 'kubernetes-engine'
  project: estafette-production
  cluster: production-europe-west4
  region: europe-west4
  zone: europe-west4-d
  serviceAccountKeyfile: 'estafette.secret(***)'
  defaults:
    namespace: production
    autoscale:
      min: 3
      max: 500
    hosts:
    - ${ESTAFETTE_GIT_NAME}.estafette.io
- name: slack-webhook-estafette
  type: slack-webhook
  workspace: estafette
  webhook: estafette.secret(***)
```

Notes:

* You can best see how much flexibility the credential type allows in the `kubernetes-engine` example where `defaults` for the `extensions/gke` image are provided. This allows you for example to override some of the extensions defaults for specific environments in case you want to deviate from those.

### Trusted images

In order to gain access to the centrally stored credentials and optionally gain additional elevated permissions _trusted images_ can be configured. For a trusted image the `path` provides the full container name except for it's tag. To automatically get access all credentials of one or multiple types each type of those credentials can be listed in the `injectedCredentialTypes` array.

Those injected credentials are then automatically mounted into the stage container(s) that use the trusted image. It mounts credentials at `/credentials/<credential type in lower snake case>.json` (for example `/credentials/kubernetes_engine.json`) in JSON format.

A typical configuration looks like:

```yaml
trustedImages:
- path: extensions/git-clone
  injectedCredentialTypes:
  - bitbucket-api-token
  - github-api-token
- path: extensions/docker
  runDocker: true
  injectedCredentialTypes:
  - container-registry
- path: extensions/gke
  injectedCredentialTypes:
  - kubernetes-engine
- path: extensions/bitbucket-status
  injectedCredentialTypes:
  - bitbucket-api-token
- path: extensions/github-status
  injectedCredentialTypes:
  - github-api-token
- path: extensions/slack-build-status
  injectedCredentialTypes:
  - slack-webhook
- path: estafette/estafette-ci-builder
  runPrivileged: true
```

Notes:

* The `bitbucket-api-token` or `github-api-token` credential types are set on the fly when a build/release job is started for either a Bitbucket or Github repository. This allows a repository to be cloned without needing to set up deploy keys or other forms of ssh authentication.