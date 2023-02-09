.. _minio-authenticate-using-dex:

============================================
Configure MinIO for Authentication using Dex
============================================

.. default-domain:: minio


.. contents:: Table of Contents
   :local:
   :depth: 1

Overview
--------

This procedure configures MinIO to use Dex as an external IDentity Provider (IDP) for authentication of users via the OpenID Connect (OIDC) protocol.

This procedure specifically covers the following steps:

.. cond:: k8s

   - Configuring Dex for use with MinIO authentication and authorization
   - Configuring a new or existing MinIO Tenant to use Dex as the OIDC provider
   - Create policies to control access of Dex-authenticated users
   - Log into the MinIO Tenant Console using SSO and a Dex-managed identity
   - Generate temporary S3 access credentials using the ``AssumeRoleWithWebIdentity`` Security Token Service (STS) API

.. cond:: not k8s

   - Configuring Dex for use with MinIO authentication and authorization
   - Configuring a new or existing MinIO cluster to use Dex as the OIDC provider
   - Create policies to control access of Dex-authenticated users
   - Log into the MinIO Tenant Console using SSO and a Dex-managed identity
   - Generate temporary S3 access credentials using the ``AssumeRoleWithWebIdentity`` Security Token Service (STS) API


This procedure was written and tested against Dex VERSION. 
The provided instructions may work against other Dex versions.

Prerequisites
-------------

.. cond:: k8s

   MinIO Kubernetes Operator and Plugin
   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

   .. include:: /includes/k8s/common-operator.rst
      :start-after: start-requires-operator-plugin
      :end-before: end-requires-operator-plugin

   MinIO Tenant
   ~~~~~~~~~~~~

   This procedure assumes your Kubernetes cluster has sufficient resources to  :ref:`deploy a new MinIO Tenant <minio-k8s-deploy-minio-tenant>`.

   You can also use this procedure as guidance for modifying an existing MinIO Tenant to enable Dex Identity Management.

.. cond:: linux or container or macos or windows

   MinIO Deployment
   ~~~~~~~~~~~~~~~~

   This procedure assumes an existing MinIO cluster running the :minio-git:`latest stable MinIO version <minio/releases/latest>`. 
   Defer to the :ref:`minio-installation` for more complete documentation on new MinIO deployments.

   This procedure *may* work as expected for older versions of MinIO.

Dex Deployment and Realm Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This procedure assumes an existing Dex deployment to which you have administrative access.
Specifically, you must have permission to create and configure Realms, Clients, Client Scopes, Realm Roles, Users, and Groups on the Dex deployment.

.. cond:: k8s

   For Dex deployments within the same Kubernetes cluster as the MinIO Tenant, this procedure assumes bidirectional access between the Dex and MinIO pods/services.

   For Dex deployments external to the Kubernetes cluster, this procedure assumes an existing Ingress, Load Balancer, or similar Kubernetes network control component that manages network access to and from the MinIO Tenant.

.. cond:: not k8s

   This procedure assumes bidirectional access between the Dex and MinIO deployments.

Install and Configure ``mc`` with Access to the MinIO Cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This procedure uses :mc:`mc` for performing operations on the MinIO cluster. 
Install ``mc`` on a machine with network access to the cluster.

.. cond:: k8s

   Your local host must have access to the MinIO Tenant, such as through Ingress, a Load Balancer, or a similar Kubernetes network control component.

See the ``mc`` :ref:`Installation Quickstart <mc-install>` for instructions on downloading and installing ``mc``.

   This procedure assumes a configured :mc:`alias <mc alias>` for the MinIO cluster. 