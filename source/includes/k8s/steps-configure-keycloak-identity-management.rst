1) Configure or Create a Client for Accessing Keycloak
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Authenticate to the Keycloak :guilabel:`Administrative Console` and navigate to :guilabel:`Clients`.

Select :guilabel:`Create client` and follow the instructions to create a new Keycloak client for MinIO.

- Set :guilabel:`Client ID` to a unique identifier for MinIO (``minio``)
- Set :guilabel:`Client type` is ``OpenID Connect``
- Set :guilabel:`Always display in console` to ``On``
- Set :guilabel:`Client authentication` to ``On``
- Ensure :guilabel:`Authentication flow` includes ``Standard flow``
- (Optional) Ensure :guilabel:`Authentication flow` includes ``Direct access grants`` (API testing)

Once created, change the provided example values as necessary to support your preferred Keycloak Realm URLs and settings:

- Set :guilabel:`Root URL` to ``${authBaseUrl}``
- Set :guilabel:`Home URL` to the Realm you want MinIO to use (``/realms/master/account/``)
- Set :guilabel:`Valid Redirect URI` to ``*``
- From :guilabel:`Keys` set :guilabel:`Use JWKS URL` to ``On``
- From :guilabel:`Advanced`, select :guilabel:`Advanced Settings` and set :guilabel:`Access Token Lifespan` to ``1 Hour``.

2) Create Client Scope for MinIO Client
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Client scopes allow Keycloak to map user attributes as part of the Java Web Token (JWT) returned in authentication requests.
This allows MinIO to reference those attributes when assigning policies to the user.
This step creates the necessary client scope to support MinIO authorization after successful Keycloak authentication.

Navigate to the :guilabel:`Client scopes` view and create a new client scope for MinIO authorization:

- Set :guilabel:`Name` to any recognizable name for the policy (``minio-authorization``)
- Ensure :guilabel:`Include in token scope` is ``On``

Once created, select the scope from the list and navigate to :guilabel:`Mappers`.

Select :guilabel:`Configure a new mapper` to create a new mapping:

- Select :guilabel:`User Attribute` as the Mapper Type
- Set :guilabel:`Name` to any recognizable name for the mapping (``minio-policy-mapper``)
- Set :guilabel:`User Attribute` to ``policy``
- Set :guilabel:`Token Claim Name` to ``policy``
- Set :guilabel:`Claim JSON Type` to ``String``
- Set :guilabel:`Multivalued` to ``On`` - this allows users to inherit any ``policy`` set in their Groups
- Set :guilabel:`Aggregate attribute values` to ``On`` - this allows users to inherit any ``policy`` set in their Groups

Once created, assign the Client Scope to the MinIO client.
Navigate to :guilabel:`Clients` and select the MinIO client.

- Select :guilabel:`Client scopes`, then select :guilabel:`Add client scope`.
- Select the previously created scope and set the :guilabel:`Assigned type` to ``default``.

3) Apply the Necessary Attribute to Keycloak Users/Groups
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You must assign an attribute named ``policy`` to the Keycloak Users or Groups. 
Set the value to to any :ref:`policy <minio-policy>` on the MinIO deployment.

For Users, navigate to :guilabel:`Users` and select or create the User:

- From :guilabel:`Credentials`, set the user password to a permanent value if not already set
- From :guilabel:`Attributes`, create a new attribute with key ``policy`` and value of any :ref:`policy <minio-policy>` (``consoleAdmin``)

For Groups, navigate to :guilabel:`Groups` and select or create the Group:

- From :guilabel:`Attributes`, create a new attribute with key ``policy`` and value of any :ref:`policy <minio-policy>` (``consoleAdmin``)

You can assign users to groups such that they inherit the specified ``policy`` attribute.
If you set the Mapper settings to enable :guilabel:`Aggregate attribute values`, Keycloak includes the aggregated array of policies as part of the authenticated user's JWT token.
MinIO can use this list of policies when authorizing the user.

You can test the configured policies of a user by using the Keycloak API:

.. code-block:: shell
   :class: copyable

   curl -d "client_id=minio" \
        -d "client_secret=secretvalue" \
        -d "grant_type=password" \
        -d "username=minio-user-1" \
        -d "password=minio-user-1-password" \
        http://keycloak-url:port/realms/REALM/protocol/openid-connect/token

If successful, the ``access_token`` contains the JWT necessary to use the MinIO :ref:`minio-sts-assumerolewithwebidentity` STS API and generate S3 credentials.

You can use a JWT decoder to review the payload an ensure it contains the ``policy`` key with one or more MinIO policies listed.

4) Configure MinIO for Keycloak Authentication
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO supports multiple methods for configuring Keycloak authentication:

- Using the MinIO Operator Console
- Using the MinIO Tenant Console
- Using a terminal/shell and the :mc:`mc admin idp openid` command

.. tab-set::

   .. tab-item:: MinIO Operator Console

      You can use the MinIO Operator Console to configure Keycloak as the External Identity Provider for the MinIO Tenant:

      .. include:: /includes/common/common-k8s-connect-operator-console.rst

      Select :guilabel:`Identity Provider` from the left-hand navigation bar, then select :guilabel:`OpenID`.
      Select :guilabel:`Create Configuration` to create a new configuration.

      Enter the following information into the modal:

      - For :guilabel:`Config URL`, specify the path to the ``.well-known/openid-configuration`` URL for your Keycloak server.
        For example, ``https://keycloak-url:port/realms/REALM/.well-known/openid-configuration``.

        If the Keycloak service is located on the same Kubernetes cluster as the MinIO Tenant, you can specify the Kubernetes DNS name for that service as the URL.

        Ensure the ``REALM`` matches the Keycloak realm you want to use for authenticating users to MinIO
      - For :guilabel:`Client ID`, specify the name of the Keycloak client created in Step 1
      - For :guilabel:`Client Secret`, specify the secret credential value for the Keycloak client created in Step 1
      - For :guilabel:`Scopes`, specify an OpenID scopes you want to include in the JWT, such as ``preferred_username`` or ``email``
        You can reference these scopes using supported OpenID policy variables for the purpose of programmatic policy configurations

      - For MinIO Tenants only accessible from a load balancer or proxy, you may need to set :guilabel:`Redirect URI` to the hostname for the MinIO Tenant Console.

        For example, if Keycloak can only access the MinIO Tenant from an Ingress or Load Balancer endpoint, specify that endpoint with ``/oauth_callback`` as the suffix.
        
        You can otherwise leave this field blank and allow MinIO to automatically determine the appropriate redirect URI to send based on the hostname from which the Console login attempt originated.

      Select :guilabel:`Save` to save the configuration.

   .. tab-item:: MinIO Tenant Console

      You can use the MinIO Tenant Console to configure Keycloak as the External Identity Provider for the MinIO Tenant.

      Access the Console service using the Node Port, Ingress, or Load Balancer endpoint.
      You can use the following command to review the Console configuration:

      .. code-block:: shell
         :class: copyable

         kubectl describe svc/TENANT_NAME-console -n TENANT_NAMESPACE

      Replace ``TENANT_NAME`` and ``TENANT_NAMESPACE`` with the name of the MinIO Tenant and it's Namespace, respectively.

      Log in as a user with administrative privileges for the MinIO deployment, such as a user with the :userpolicy:`consoleAdmin` policy.

      Select :guilabel:`Identity` from the left-hand navigation bar, then select :guilabel:`OpenID`.
      Select :guilabel:`Create Configuration` to create a new configuration.

      Enter the following information into the modal:

      - For :guilabel:`Name`, enter a unique name for the Keycloak instances 
      - For :guilabel:`Config URL`, specify the path to the ``.well-known/openid-configuration`` URL for your Keycloak server.
        For example, ``https://keycloak-url:port/realms/REALM/.well-known/openid-configuration``

        Ensure the ``REALM`` matches the Keycloak realm you want to use for authenticating users to MinIO
      - For :guilabel:`Client ID`, specify the name of the Keycloak client created in Step 1
      - For :guilabel:`Client Secret`, specify the secret credential value for the Keycloak client created in Step 1
      - For :guilabel:`Display Name`, specify the user-facing name the MinIO Console displays as part of the Single-Sign On (SSO) workflow for the configured Keycloak service
      - For :guilabel:`Scopes`, specify an OpenID scopes you want to include in the JWT, such as ``preferred_username`` or ``email``
        You can reference these scopes using supported OpenID policy variables for the purpose of programmatic policy configurations

      - For MinIO deployments only accessible from a load balancer or proxy, you may need to set :guilabel:`Redirect URI` to the hostname for the MinIO Console.
        
        You can otherwise leave this field blank and allow MinIO to automatically determine the appropriate redirect URI to send based on the hostname from which the Console login attempt originated.

      Select :guilabel:`Save` to save the configuration.

   .. tab-item:: CLI

      You can use the :mc-cmd:`mc admin idp openid add` command to create a new configuration for the Keycloak service.
      The command takes all supported :ref:`OpenID Configuration Settings <minio-open-id-config-settings>`.

      The following example assumes an :ref:`alias <alias>` which corresponds to the MinIO Tenant.

      .. code-block:: shell
         :class: copyable

         mc admin idp openid add ALIAS keycloak \
            client_id=MINIO_CLIENT \
            client_secret=MINIO_CLIENT_SECRET \
            config_url="https://keycloak-url:9090/realms/REALM/.well-known/openid-configuration" \
            display_name="SSO_IDENTIFIER"
            scopes="openid,email,preferred_username" \
            redirect_uri="https://minio-console-url:9001/oauth_callback"

      - Replace ``keycloak`` with a unique identifier for this Keycloak configuration.

      - Replace ``MINIO_CLIENT`` and ``MINIO_CLIENT_SECRET`` with the Keycloak client ID and secret configured in Step 1

      - Replace ``config_url`` with the path to the ``.well-known/openid-configuration`` URL for your Keycloak server

      - Replace ``display_name`` with a user-facing name the MinIO Console displays as part of the Single-Sign On (SSO) workflow for the configured Keycloak service

      - Replace ``scopes`` with the OpenID scopes you want to include in the JWT, such as ``preferred_username`` or ``email``

      - For MinIO deployments only accessible from a load balancer or proxy, you may need to set ``redirect_uri`` to the hostname for the MinIO Console. 
        You can otherwise omit this field and direct MinIO to determine the appropriate URI based on the hostname from which the Console login attempt originated.

You must restart the MinIO deployment for the changes to apply.

Check the MinIO logs and verify that startup succeeded with no errors related to the OIDC configuration.

If you attempt to log in with the Console, you should now see a (SSO) button using the configured :guilabel:`Display Name`.

Specify a configured user and attempt to log in.
MinIO should automatically redirect you to the Keycloak login entry.
Upon successful authentication, Keycloak should redirect you back to the MinIO Console using either the originating Console URL *or* the :guilabel:`Redirect URI` if configured.

5) Generate Application Credentials using the Security Token Service (STS)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Applications using an S3-compatible SDK must specify credentials in the form of a access key and secret key.
The MinIO :ref:`minio-sts-assumerolewithwebidentity` API returns the necessary temporary credentials, including a required session token, using a JWT returned by Keycloak after authentication.

You can test this workflow using the following sequence of HTTP calls and the ``curl`` utility:

1. Authenticate as a Keycloak user and retrieve the JWT token

   .. code-block:: shell
      :class: copyable

      curl -X POST "https://keycloak-url:port/realms/REALM/protocol/openid-connect/token" \
           -H "Content-Type: application/x-www-form-urlencoded" \
           -d "username=USER" \
           -d "password=PASSWORD" \
           -d "grant_type=password" \
           -d "client_id=CLIENT" \
           -d "client_secret=SECRET"

   - Replace the ``USER`` and ``PASSWORD`` with the credentials of a Keycloak user on the ``REALM``.
   - Replace the ``CLIENT`` and ``SECRET`` with the client ID and secret for the MinIO-specific Keycloak client on the ``REALM``

   You can process the results using ``jq`` or a similar JSON-formatting utility.
   Extract the ``access_token`` field to retrieve the necessary access token.
   Pay attention to the ``expires_in`` field to note the number of seconds before the token expires.

2. Generate MinIO Credentials using the ``AssumeRoleWithWebIdentity`` API

   .. code-block:: shell
      :class: copyable

      curl -X POST "https://minio-url:port" \
           -H "Content-Type: application/x-www-form-urlencoded" \
           -d "Action=AssumeRoleWithWebIdentity" \
           -d "Version=2011-06-15" \
           -d "DurationSeconds=86000" \
           -d "WebIdentityToken=TOKEN"

   Replace the ``TOKEN`` with the ``access_token`` value returned by Keycloak.

   The API should return an XML document on success containing the following keys:
   
   - ``Credentials.AccessKeyId`` - the Access Key for the Keycloak User
   - ``Credentials.SecretAccessKey`` - the Secret Key for the Keycloak User
   - ``Credentials.SessionToken`` - the Session Token for the Keycloak User
   - ``Credentials.Expiration`` - the Expiration Date for the generated credentials.

3. Test the Credentials

   Use your preferred S3-compatible SDK to connect to MinIO using the generated credentials.

   For example, the following Python code using the MinIO :ref:`Python SDK <minio-python-quickstart>` connects to the MinIO deployment and returns a list of buckets:

   .. code-block:: python

      from minio import Minio

      client = MinIO(
         "minio-url:9000",
         access_key = "ACCESS_KEY",
         secret_key = "SECRET_KEY",
         session_token = "SESSION_TOKEN"
         secure = True
      )

      client.list_buckets()

6) Next Steps
~~~~~~~~~~~~~

Applications should implement the STS flow using their SDK of choice.
When STS credentials expire, applications should have logic in place to regenerate the JWT token, STS token, and MinIO credentials before retrying and continuing operations.

Alternatively, users can generate :ref:`access keys <minio-id-access-keys>` through the MinIO Console for the purpose of creating long-lived API-key like access using their Keycloak credentials.