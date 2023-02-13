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

## Deploying OpenSearch

There are two different ways to deploy OpenSearch to Kubernetes:
[The OpenSearch Operator](https://opensearch.org/docs/2.5/tools/k8s-operator/) or through
using the [OpenSearch Helm Chart](https://github.com/opensearch-project/helm-charts/tree/main/charts/opensearch).
This guide will provide the steps for a helm-based deployment.
