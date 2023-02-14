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

## References

- Keycloak OpenSearch - https://github.com/cht42/opensearch-keycloak
- OpenID Backend Documentation - https://opensearch.org/docs/latest/security/authentication-backends/openid-connect/
- Configuring OpenSearch 2.x With OpenID - https://www.rushworth.us/lisa/?p=9370
