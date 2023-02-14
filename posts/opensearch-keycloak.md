---
title: Connecting OpenSearch to Keycloak
description: ''
tags:
  - kubernetes
  - devops
  - tutorial
  - elastic
cover_image: ''
canonical_url: null
published: false
id: 1363303
---

## Managing Certificates

Generally speaking it is good practice to secure communication with
your database. Before we can deploy OpenSearch, we would first have
to create certificates. [cert-manager](https://cert-manager.io) is a helpful tool which can be deployed
onto Kubernetes to create and manage certificates.
With our applications being disbursed across several different
namespaces, creating certificates will take place across multiple
stages:

- Deploy Cert Manager
- Deploy Trust Manager
- Deploy the root certificate issuer
- Generate a root CA using the root issuer
- Create a cluster-issuer which will enable us to deploy certs in other namespaces

[trust-manager](https://cert-manager.io/docs/projects/trust-manager) will
enable us to share the root CA chain globally across all the different
namespaces so that we can use the public certificates to verify any
certificate which is derived from the root CA.

The following Ansible snippet details how to preform these steps.

```yml
- name: Deploy cert-manager
  kubernetes.core.helm:
    release_name: cert-manager
    release_namespace: cert-manager
    create_namespace: true
    wait: true
    chart_ref: cert-manager
    chart_repo_url: https://charts.jetstack.io
    values:
      installCRDs: true
- name: Deploy trust-manager
  kubernetes.core.helm:
    release_name: cert-manager-trust
    release_namespace: cert-manager
    create_namespace: true
    wait: true
    chart_ref: cert-manager-trust
    chart_repo_url: https://charts.jetstack.io
    values:
      installCRDs: true
- name: Create Root Issuer
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: cert-manager.io/v1
      kind: Issuer
      metadata:
        namespace: cert-manager
        name: root-issuer
      spec:
        # Example uses self-signed
        # It is advisable to utilize a different kind of Issuer
        # for production
      selfSigned: {}
- name: Create Root CA Certs
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: cert-manager.io/v1
      kind: Certificate
      metadata:
        namespace: cert-manager
        name: root-ca
      spec:
        isCA: true
        # Omitted multiple values.
        # https://cert-manager.io/docs/usage/certificate/
        # for full Certificate spec
        secretName: root-ca
        privateLey:
          algorithm: ECDSA
          size: 256
        issuerRef:
          name: root-issuer
          kind: Issuer
          group: cert-manager.io
- name: Create Cluster Issuer
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: cert-manager.io/v1
      kind: ClusterIssuer
      metadata:
        name: cluster-issuer
        namespace: cert-manager
      spec:
        ca:
          secretName: root-ca
- name: Create Root CA Chain bundle
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: trust.cert-manager.io/v1alpha1
      kind: Bundle
      metadata:
        name: root-bundle
        namespace: cert-manager
      spec:
        sources:
          - secret:
              name: root-ca
              key: ca.crt
        target:
          configMap:
            key: root.ca.crt
```

## Deploying Keycloak

Before we can authenticate OpenSearch against Keycloak, we'll need to
install Keycloak. The following Ansible snippet demonstrates how to
deploy Keycloak onto a Kubernetes cluster using the [codecentric helm chart](https://github.com/codecentric/helm-charts/tree/master/charts/keycloak).

```yml
- name: Deploy Keycloak
  kubernetes.core.helm:
    wait: true
    release_name: keycloak
    release_namespace: keycloak
    chart_ref: keycloak
    chart_repo_url: https://codecentric.github.io/helm-charts
    values:
      extraEnv: |
        - name: KEYCLOAK_USER
          value: {{ keycloak_user }}
        - name: KEYCLOAK_PASSWORD
          value: {{ keycloak_password }}
        - name: PROXY_ADDRESS_FORWARDING
          value: "true"
- name: Provision admin role
  community.general.keycloak_role:
    auth_keycloak_url: "{{ keycloak_url }}"
    auth_realm: master
    auth_username: "{{ keycloak_user }}"
    auth_password: "{{ keycloak_password }}"
    validate_certs: false
    realm: "{{ realm_name }}"
    name: admin
    description: >-
      The admin role enables administrator privileges for any
      user which assigned to this role.
      The admin role will also map to the admin role in OpenSearch
- name: Provision readall role
  community.general.keycloak_role:
    auth_keycloak_url: "{{ keycloak_url }}"
    auth_realm: master
    auth_username: "{{ keycloak_user }}"
    auth_password: "{{ keycloak_password }}"
    validate_certs: false
    realm: "{{ realm_name }}"
    name: admin
    description: >-
      The readall role enables read-only privileges.
      readall will have read-only privileges in OpenSearch
```

Sections pertaining to the ingress and associated rules were left out
to keep the code snippet small. The above snippet provisions two
roles on our Keycloak IDp which will map directly to OpenSearch roles
with the same names.

A client will also need to be prepared so that OpenSearch may
authenticate with Keycloak as an OpenID backend. The below Ansible
snippet demonstrates how to set up the client using the
`community.general.keylcloak_client` plugin.

```yml
- name: Deploy OpenSearch Client
  community.general.keycloak_client:
    auth_keycloak_url: "{{ keycloak_url }}"
    auth_realm: master
    validate_certs: false
    auth_username: "{{ keycloak_user }}"
    auth_password: "{{ keycloak_password }}"
    client_id: opensearch-dashboards
    secret: "{{ os_client_secret }}"
    realm: "{{ realm_name }}"
    enabled: true
    direct_access_grants_enabled: true
    authorization_services_enabled: true
    service_accounts_enabled: true
    redirect_uris:
      # localhost:5601 is used as the OpenSearch-Dashboards URL
      - https://localhost:5601/*
    protocol_mappers:
      - name: opensearch-audience
        protocol: openid-connect
        protocolMapper: oidc-audience-mapper
        config:
          id.token.claim: "true"
          access.token.claim: "true"
          included.client.audience: opensearch-dashboards
      - name: opensearch-role-mapper
        protocol: openid-connect
        protocolMapper: oidc-usermodel-realm-role-mapper
        config: {
          "multivalued": "true",
          "userinfo.token.claim": "true",
          "id.token.claim": "true",
          "access.token.claim": "true",
          "claim.name": "roles",
          "jsonType.label": "String"
        }
```

`direct_access_grants_enabled` enables Keycloak direct access grants
which allows the client to authenticate users by directly using
username/password combos.

`service_accounts_enabled` is for future use which will enable the
Keycloak client to use service accounts (i.e. to authenticate applications).

`authorization_services_enabled` enables fine-grained authorization
support for a client.

In the `protocol_mappers` section, two protocol mappers are created.
The protocol mappers are utilized to map different items to fields on
the JWT which is returned once a user logs in. For OpenSearch
OpenID authentication, OpenSearch looks for a key to identify the
roles associated with a user account to map to an OpenSearch role
which will determine which functions that the user is allowed to access.

I'm not exactly sure if the audience-mapper is needed, but I left it
in there since I've used it for other authentication purposes
(i.e. express-passport)

![OpenSearch Keycloak Client](./assets/Images/keycloak-opensearch/Screen%20Shot%202023-02-13%20at%207.02.34%20PM.png)
![OpenSearch Audience Mapper](./assets/Images/keycloak-opensearch/Screen%20Shot%202023-02-13%20at%207.02.40%20PM.png)
![OpenSearch Role Mapper](./assets/Images/keycloak-opensearch/Screen%20Shot%202023-02-13%20at%207.02.47%20PM.png)

## Deploying OpenSearch

There are two different ways to deploy OpenSearch to Kubernetes:
[The OpenSearch Operator](https://opensearch.org/docs/2.5/tools/k8s-operator/) or through using the [OpenSearch Helm Chart](https://github.com/opensearch-project/helm-charts/tree/main/charts/opensearch).
This guide will provide the steps for a helm-based deployment.

There are two main configuration files which connect OpenSearch to an
OpenID provider, in this case, Keycloak: `opensearch-security/config.yml`
and `opensearch-dashboards.yml`.
`opensearch-security/config.yml` connects the OpenSearch security plugin
and `opensearch_dashboards.yml` connects the OpenSearch Dashboards UI.

`opensearch-security/config.yml`

```yml
_meta:
  type: "config"
  config_version: 2
config:
  dynamic:
    authz: {}
    authc:
      basic_internal_auth_domain:
        http_enabled: true
        transport_enabled: true
        order: 1
        http_authenticator:
          type: basic
          challenge: false
        authentication_backend:
          type: intern

      openid_auth_domain:
        http_enabled: true
        transport_enabled: true
        order: 0
        http_authenticator:
          type: openid
          challenge: false
          config:
            openid_connect_idp:
              enable_ssl: true
              verify_hostnames: false
              pemtrustedcas_filepath: /usr/share/opensearch/config/root-ca/root.ca.crt
            subject_key: preferred_username
            roles_key: roles
            openid_connect_url: "{{ keycloak_url }}/auth/realms/pantry/.well-known/openid-configuration"
        authentication_backend:
          type: noop
```

`_meta` contains the metadata configuration which is boilerplate
designations for the file as a configuration file. `config` contains
the configuration that the OpenSearch Security plugin will use to
configure how authentication and authorization will be managed in
OpenSearch.

`authc` contains [authentication backends](https://opensearch.org/docs/latest/security/authentication-backends/authc-index/).
The authentication backends which are provided by the configuration
snippet are basic, and openid. The basic authentication is used for the
built-in users (i.e. admin, kibanaserver).
The openid backend is used to connect to Keycloak.

`openid_connect_url` points to the OpenID provider URL. OpenSearch
expects the URL to point to a `.well-known/openid-configuration` endpoint
which is used to fetch the metadata configuration.

`roles_key` points to the field on the JWT which is issued by the
OpenID provider where the realm roles are set. The roles are used to
map the OpenID realm roles to OpenSearch roles for fine-grained
access control.

`subject_key` points to the field on the JWT which is issued by the
OpenID provider where the username is located. This field is usually
`username`, `preferred_username`, or `email`.

`openid_connect_idp` contains the details which are used to provide
TLS/SSL values to the OpenID server. `enable_ssl` tells OpenSearch
that the OpenID connection is over SSL. `verify_hostnames` instructs
OpenSearch whether or not to verify the HTTP hostname against the
hostname provided by the certificate. `pemtrustedcas_filepath`
contains the file path to the root Certificate Authority certificate file
which is used to verify the certificate. Certificate verification prevents
man-in-the-middle attacks or certificate impersonation.

Based on the OpenSearch documentation, setting the `authentication_backend`
to `noop` is required because JSON web-tokens already contain the
required information to verify the request.

`opensearch_dashboards.yml`

```yml
server:
  name: dashboards
  host: 0.0.0.0
  ssl:
    enabled: true
    key: /usr/share/dashboards/certs/tls.key
    certificate: /usr/share/dashboards/certs/tls.crt
opensearch_security:
  auth.type: openid
  openid:
    connect_url: https://{{ domain_name }}/auth/realms/pantry/.well-known/openid-configuration
    base_redirect_url: https://localhost:5601
    client_id: opensearch-dashboards
    client_secret: {{ os_client_secret }}
    scope: openid profile email
    header: Authorization
    verify_hostnames: false
    root_ca: /usr/share/dashboards/root-ca/root.ca.crt
    trust_dynamic_headers: "true"
opensearch:
  requestHeadersAllowlist: ["Authorization", "security_tenant"]
  hosts: [ "opensearch-cluster-master" ]
  username: "kibanaserver"
  password: "kibanaserver"
  ssl:
    certificateAuthorities: /usr/share/dashboards/certs/ca.crt
```

`client_id` contains the OpenID client ID which OpenSearch Dashboards
will use to authenticate with Keycloak.
`client_secret` contains the client secret which OpenSearch Dashboards
will use to authenticate with Keycloak.
`root_ca` contains the root CA path for OpenSearch Dashboards to verify
the OpenID provider (Keycloak) certificate.
`verify_hostnames` instructs OpenSearch whether or not to verify the
HTTP hostname against the hostname provided by the certificate.
`scope` contains the scopes which are used to determine the identity
from the token issued by the OpenID provider.
`auth.type` instructs OpenSearch Dashboards to use OpenID for user logins.

Refer to the [OpenSearch](https://github.com/opensearch-project/helm-charts/blob/main/charts/opensearch/values.yaml) and the [OpenSearch Dashboards](https://github.com/opensearch-project/helm-charts/blob/main/charts/opensearch-dashboards/values.yaml) for all the supported values for OpenSearch and OpenSearch Dashboards deployment.

The configuration in `opensearch-security/config.yml` will be added to
the OpenSearch helm chart like so:

```yml
- name: Deploy OpenSearch
  kubernetes.core.helm:
    release_name: opensearch
    release_namespace: opensearch
    chart_ref: opensearch
    chart_repo_url: https://opensearch-project.github.io/helm-charts
    values:
      replicas: 1
      minimumMasterNodes: 0
      securityConfig:
        enabled: true
        config:
          dataComplete: false
          data:
            config.yml: |-
              {{ opensearch_security_config }}
```

I've omitted values for `opensearch.yml`, volumes, and volume mounts.
Volumes and volume mounts will be required for security configuration
to mount the SSL certificates.

The configuration in `opensearch_dashboards.yml` can be added to the
opensearch-dashboards release like so:

```yml
- name: Deploy OpenSearch Dashboards
  kubernetes.core.helm:
    release_name: opensearch-dashboards
    release_namespace: opensearch
    chart_ref: opensearch-dashboards
    chart_repo_url: https://opensearch-project.github.io/helm-charts
    values:
      config:
        opensearch_dashboards.yml: |-
        {{ opensearch_dashboards_config }}
```

Once the deployments are complete, Dashboards should be accessible
through Kubernetes port forwarding

```bash
kubectl port-forward services/opensearch-dashboards 5601:5601 -n opensearch
```

The backends are visible through the OpenSearch Security page.

![Dashboards Backends](./assets/Images/keycloak-opensearch/Screen%20Shot%202023-02-13%20at%207.07.56%20PM.png)

## References

- Keycloak OpenSearch - https://github.com/cht42/opensearch-keycloak
- OpenID Backend Documentation - https://opensearch.org/docs/latest/security/authentication-backends/openid-connect/
- Configuring OpenSearch 2.x With OpenID - https://www.rushworth.us/lisa/?p=9370
- OpenSearch Dashboards helm chart - https://github.com/opensearch-project/helm-charts/blob/main/charts/opensearch-dashboards/values.yaml
- OpenSearch helm chart - https://github.com/opensearch-project/helm-charts/blob/main/charts/opensearch/values.yaml
