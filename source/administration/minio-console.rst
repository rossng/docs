.. _minio-console:

=============
MinIO Console
=============

.. default-domain:: minio

.. contents:: Table of Contents
   :local:
   :depth: 2


The MinIO Console is a rich graphical user interface that provides similar functionality to the :mc:`mc` command line tool.

.. image:: /images/minio-console/minio-console.png
   :width: 600px
   :alt: MinIO Console Landing Page provides a view of the Object Browser for the authenticated user
   :align: center

This page provides an overview of the MinIO Console and describes configuration options and instructions for logging in.

Overview
--------

You can use the MinIO Console for administration tasks like Identity and Access Management, Metrics and Log Monitoring, or Server Configuration.

The MinIO Console is embedded as part of the MinIO Server. 
You can also deploy a standalone MinIO Console using the instructions in the :minio-git:`github repository <console>`.

Configuration
-------------

The MinIO Console inherits the majority of its configuration settings from the
MinIO Server. The following environment variables enable specific behavior in
the MinIO Console:

.. list-table::
   :header-rows: 1
   :widths: 30 70
   :width: 100%

   * - Environment Variable
     - Description

   * - :envvar:`MINIO_PROMETHEUS_URL`
     - The URL for a Prometheus server configured to scrape metrics from the 
       MinIO deployment. The MinIO Console uses this server for populating the
       metrics dashboard.

       See :ref:`minio-metrics-collect-using-prometheus` for a tutorial on 
       configuring Prometheus to collect metrics from MinIO.

   * - :envvar:`MINIO_SERVER_URL`
     - The URL hostname the MinIO Console uses for connecting to the MinIO 
       Server. The hostname *must* be resolveable and reachable for the
       Console to function correctly.

       The MinIO Console connects to the MinIO Server using an IP 
       address by default. For example, when the MinIO Server starts up, 
       the server logs include a line 
       ``API: https://<IP ADDRESS 1> https://<IP ADDRESS 2>``.
       The MinIO Console defaults to connecting using ``<IP ADDRESS 1>``.

       The MinIO Console may require setting this variable in the following scenarios:
       
       - The MinIO server TLS certificates do not include the local IP address
         as a :rfc:`Subject Alternative Name <5280#section-4.2.1.6>` (SAN). 
         Specify a hostname contained in the TLS certificate to allow the MinIO 
         Console to validate the TLS connection.

       - The MinIO server's local IP address is not reachable by the MinIO
         Console. Specify a resolveable hostname for the MinIO Server.

       - A load balancer or reverse proxy controls traffic to the MinIO server,
         such that the MinIO Console cannot reach the server without going
         through the load balancer/proxy. Specify the load balancer/proxy 
         URL for the MinIO server.

   * - :envvar:`MINIO_BROWSER_REDIRECT_URL`
     - The externally resolvable hostname for the MinIO Console used by the 
       configured :ref:`external identity manager 
       <minio-authentication-and-identity-management>` for returning the
       authentication response.

       This variable is typically necessary when using a reverse proxy, 
       load balancer, or similar system to expose the MinIO Console to the 
       public internet. Specify an externally reachable hostname that resolves
       to the MinIO Console.

Static vs Dynamic Port Assignment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO by default selects a random port for the MinIO Console on each server
startup. Browser clients accessing the MinIO Server are automatically 
redirected to the MinIO Console on its dynamically selected port. 
This behavior emulates the legacy web browser behavior while reducing the
the risk of a port collision on systems which were running MinIO *before* the 
embedded Console update.

You can select an explicit static port by passing the 
:mc-cmd:`minio server --console-address` commandline option when starting 
each MinIO Server in the deployment. 

For example, the following command starts a distributed MinIO deployment using
a static port assignment of ``9001`` for the MinIO Console. This deployment
would respond to S3 API operations on the default MinIO server port ``:9000``
and browser access on the MinIO Console port ``:9001``.

.. code-block:: shell
   :class: copyable

   minio server https://minio-{1...4}.example.net/mnt/drive-{1...4} \
         --console-address ":9001"

Deployments behind network routing components which require static ports for 
routing rules may require setting a static MinIO Console port. For example,
load balancers, reverse proxies, or Kubernetes ingress may by default block
or exhibit unexpected behavior with the the dynamic redirection behavior.

Logging In
----------

Logging into the MinIO Console depends on how you configured identity management for the deployment.

- When using the built-in MinIO identity management solution, the sign-in screen displays a standard login screen.
  Enter your Username and Password to log in to the MinIO Console.
- If logging in with a third party application and :ref:`MinIO's Security Token Service (STS) <minio-security-token-service>`, select :guilabel:`Use STS` and enter the Username, Secret, and Token.
- If the deployment uses a single OpenID or Active Directory/LDAP identity provider solution, select the provider's button to proceed to the login screen.
- If the deployment has multiple OpenID and/or Active Directory/LDAP identify management providers configured, the MinIO Console's sign-in screen provides a dropdown list of providers.
  Select the provider you wish to use to log in to the MinIO Console, then enter the credentials.

.. admonition:: Try out the Console using MinIO's Play testing environment
   :class: note

   You can explore the Console using https://play.min.io:9443. 
   Log in with the following credentials:

   - Username: ``Q3AM3UQ867SPQQA43P2F``
   - Password: ``zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG``

   The Play Console connects to the MinIO Play deployment at https://play.min.io.
   You can also access this deployment using :mc:`mc` and using the ``play`` alias.

Documentation
-------------

The :guilabel:`Documentation` tab opens this documentation site in a separate browser window or tab.

Available Tasks
---------------

Once logged in to the MinIO Console, users can perform many kinds of tasks.

- :ref:`Manage objects <minio-console-managing-objects>` by browsing or uploading objects, managing bucket settings, or creating tiers.
- :ref:`Review or modify identity and security <minio-console-security-access>` with access keys, policies, and Identity Provider settings.
- :ref:`Monitor the health and activities <minio-console-managing-deployment>` with metrics, notifications, or site replication
- :ref:`Manage your deployment's license and SUBNET Subscription <minio-console-subscription>`

.. toctree::
   :titlesonly:
   :hidden:

   /administration/console/managing-deployment
   /administration/console/managing-objects
   /administration/console/security-and-access
   /administration/console/subnet-registration