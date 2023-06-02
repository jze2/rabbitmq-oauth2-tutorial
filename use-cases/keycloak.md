# Use KeyCloak as OAuth 2.0 server

You are going to test 3 OAuth flows:
1. Access management ui via a browser
2. Access management rest api
3. Access AMQP protocol

## Prerequisites to follow this guide

- Docker
- make

## Deploy Key Cloak

First, deploy **Key Cloak**. It comes preconfigured with all the required scopes, users and clients.
```
make start-keycloak
```
**Key Cloak** comes configured with its own signing key. And the [rabbitmq.config](../conf/keycloak/rabbitmq.config)
used by `make start-keycloak` is also configured with the same signing key.

To access KeyCloak management interface go to http://0.0.0.0:8080/ and enter `admin` as username and password.

There is a dedicated **KeyCloak realm** called `Test` configured as follows:
- You configured an [rsa](http://0.0.0.0:8080/admin/master/console/#/realms/test/keys) signing key
- And a [rsa provider](http://0.0.0.0:8080/admin/master/console/#/realms/test/keys/providers)
- And three clients: `rabbitmq-client-code` for the rabbitmq managament ui, `mgt_api_client` to access via the
management api and `producer` to access via AMQP protocol.


## Start RabbitMQ

To start RabbitMQ run the following two commands. The first one tells RabbitMQ to pick up the
rabbit.config found under [conf/keycloak/rabbitmq.config](../conf/keycloak/rabbitmq.config)
```
export MODE=keycloak
make start-rabbitmq
```

## Access Management ui

Go to http://localhost:15672/#/, click on `Click here to log in` button, and enter the credentials
`rabbit_admin` as username and `rabbit_admin` as password. This is the single user created in keycloak with the
appropriate scopes to access the management ui.

## Access Management api

Access the management api using the client [mgt_api_client](http://0.0.0.0:8080/admin/master/console/#/realms/test/clients/c5be3c24-0c88-4672-a77a-79002fcc9a9d) which has the scope [rabbitmq.tag:administrator](http://0.0.0.0:8080/admin/master/console/#/realms/test/client-scopes/f6e6dd62-22bf-4421-910e-e6070908764c)

```
make curl-keycloak url=http://localhost:15672/api/overview client_id=mgt_api_client secret=LWOuYqJ8gjKg3D2U8CJZDuID3KiRZVDa
```

## Access AMQP protocol with PerfTest

To test OAuth2 authentication with AMQP protocol you are going to use RabbitMQ PerfTest tool which uses RabbitMQ Java Client.
First you obtain the token and pass it as a parameter to the make target `start-perftest-producer-with-token`.

```
make start-perftest-producer-with-token PRODUCER=producer TOKEN=$(bin/keycloak/token producer kbOFBXI9tANgKUq8vXHLhT6YhbivgXxn)
```

**NOTE**: Initializing an application with a token has one drawback: the application cannot use the connection beyond the lifespan of the token. See the next section where you demonstrate how to refresh the token.

## Access AMQP protocol with Pika

This section is about testing OAuth 2.0 authentication with AMQP protocol and with Pika library. And more specifically, you
are demonstrating how to refresh a token on a live AMQP connections.

You can see the Python sample application [here](../pika-client/producer.py).

To run this sample code proceed as follows:
```
python3 --version
pip install pika
pip install requests
python3 pika-client/producer.py producer kbOFBXI9tANgKUq8vXHLhT6YhbivgXxn
```
> Ensure you install pika 1.3

## Access Management UI

Go to http://localhost:15672, click on the single button on the page which redirects to **Key Cloak** to authenticate.
Enter `rabbit_admin` and `rabbit_admin` and you should be redirected back to RabbitMQ Management fully logged in.


## Stop keycloak

`make stop-keycloak`


## Notes about setting up KeyCloak

### Configure JWT signing Keys

At the realm level, you go to `Keys > Providers` tab and create one of type `rsa` and you enter the
private key and certificate of the public key. In this repository you do not have yet the certificate
for the public key but it is easy to generate. Give it priority `101` or greater than the rest of
available keys so that it is picked up when you request a token.

IMPORTANT: You cannot hard code the **kid** hence you have to add the key to rabbitmq via the command
```
docker exec -it rabbitmq rabbitmqctl add_uaa_key Gnl2ZlbRh3rAr6Wymc988_5cY7T5GuePd5dpJlXDJUk --pem-file=conf/public.pem
```
or you have to modify the RabbitMQ configuration so that it says `Gnl2ZlbRh3rAr6Wymc988_5cY7T5GuePd5dpJlXDJUk`
rather than `legacy-token-key`.

### Configure Client

For backend applications which uses **Client Credentials flow** you create a **Client** with:
- **Access Type** : `public`  
  Note: **Service Accounts Enabled** is available only for `private` (**Client authentication** `On`) in KeyCloak 20.
- **Authentication flow** : `Standard Flow`
- With **Service Accounts Enabled** `On`. If it is not enabled you do not have the tab `Credentials`
- In tab `Credentials` you have the client id secret


### Configure Client scopes

> *Default Client Scope* are scopes automatically granted to every token. Whereas *Optional Client Scope* are
scopes which are only granted if they are explicitly requested during the authorization/token request flow.


### Include appropriate aud claim

You must configure a **Token Mapper** of type **Hardcoded claim** with the value of rabbitmq's *resource_server_id**.
You can configure **Token Mapper** either to a **Client scope** or to a **Client**.

### Export Keycloak configuration
For testing purposes, once you modified keycloak configuration, you would want to export keycloak configuration.
When done, connect to the keycloak container and export your configuration before removing the container
> The following command overrides the default configuration provided with this repository
```shell
docker exec -it keycloak /opt/keycloak/bin/kc.sh export --realm test --dir /opt/keycloak/data/import/ --users realm_file
```

### Possible issue on MacOS

If you want to run this configuration on MacOS, you could have problem reaching keycloak pointing to 0.0.0.0:8080.

1. add `keycloak` entry in your `hosts` file
```shell
echo "127.0.0.1 keycloak" > /etc/hosts
```
1. modify the [rabbitmq.config](../conf/keycloak/rabbitmq.config) configuration to appropriately point to `keycloak` host
```shell
...
{oauth_provider_url, "http://keycloak:8080/realms/test"}
...
```
  
-----
  
## Example of RabbitMQ 3.11 (running in Kubernetes) / KeyCloak 20 configuration with OAUTH2 for accessing management UI via a browser

Best method of learning is by an example, so based on the above and some
forum support I add steps here when running RabbitMQ with KeyCloak 20 in
Kubernetes (I had KC already running):

### KeyCloak configuration

Note: If not specified otherwise, you can leave other settings at defaults.

1. Think out **resource server ID** that will represent your RabbitMQ server,
   for example `rabbitmq`.
2. Create a realm (not neccessary but recommended for testing to isolate your
   tests from the rest of KC's configuration), for example `rabbitmq`.
3. Go to Realm settings->Keys->Providers tab and add for example
   `rsa-generated` provider (might be already present). Then go to "Keys list"
   tab and save automatically generated Base64 encoded Certificate of the
   `rsa-generated` provider. Also save the Kid value for later use in RabbitMQ
   configuration. Convert downloaded certificate to PEM:
   ```shell
   cat certificate.txt | base64 --decode > certificate.der
   openssl x509 -inform der -in certificate.der -out certificate.pem
   ```
4. Create a client (for example `rabbitmq-web-ui`)
   - Turn on **Client authentication**
     (**Service Accounts Enabled** is not neccessary).
   - Fill fields in Access settings section, for example:  
     **Root URL**: `https://rabbitmq-new.internal.example.com`  
     **Home URL**: `https://rabbitmq-new.internal.example.com`  
     **Valid redirect URIs**: `https://rabbitmq-new.internal.example.com/*`  
     **Valid post logout redirect URIs**: `https://rabbitmq-new.internal.example.com/*`  
     **Web origins**: `+`
   - Save the **Client secret** value from "Credentials" tab for later use in
     RabbitMQ configuration.
   - In "Roles" tab create roles recognized by RabbitMQ management UI in the
     format **resource server ID**.tag:**tag**, i.e. for example:
     ```
     rabbitmq.tag:administrator
     rabbitmq.tag:management
     rabbitmq.tag:monitoring
     rabbitmq.tag:policymaker
     ```
     $`\textcolor{red}{\text{
     The roles above just specify which menu entries will be available in the
     management web UI.
     }}`$  
     $`\textcolor{red}{\text{
     They do
     }}`$
     $`\textcolor{red}{\textbf{NOT}}`$
     $`\textcolor{red}{\text{
     specify that a user will be able to perform any actions in the management
     web UI (permissions).
     }}`$  
     $`\textcolor{red}{\text{
     For that you need to specify other roles in the format
     }}`$  
     $`\textcolor{red}{\textbf{
     resource server ID}\text{.}\textbf{permission}\text{:}\textbf{vhost pattern}\text{/}\textbf{resource pattern}\text{/}\textbf{routing key pattern}\text{, for example:
     }}`$
     ```
     rabbitmq.configure:*/*/*
     rabbitmq.read:*/*/*
     rabbitmq.write:*/*/*
     ```
     Note: When you share the realm with other applications connecting to
     RabbitMQ, you will want to have roles defined as Realm roles so that
     you don't have to duplicate them for every client but I wanted to test
     other configuration possibility.
5. Create a Client scope (for example `rabbitmq-web-ui`). There we specify
   what will be included in the returned JWT token for RabbitMQ.
   - Set **Type** to `Default`.
   - View details of the created client scope and go to "Mappers" tab.
     Click Add mapper->By configuration and select **Audience**.
     Give it a name and in the **Included Custom Audience** field specify
     **resource server ID**, i.e. in our case `rabbitmq`.
     This represents the intended recipient of the generated JWT token
     (RabbitMQ requires this).
   - Since RabbitMQ does not navigate the JWT hierarchy, all fields it is
     interested in must be at the root level but KeyCloak returns role list
     deeper in the JWT hierarchy, so we need to pull it at the root level.
     Add another mapper (By configuration) and select **User Client Role**.
     Give it a name and in the **Client ID** field select our client,
     i.e. `rabbitmq-web-ui`. In the **Token Claim Name** fill the JWT field
     name that will contain all the user roles (by expanding user's groups
     defined later), for example `roles`.
   - You may add other mappers suitable for your use (like **full name**).
   - Now since your user might be member of other non-RabbitMQ related groups
     in your environment (like monitoring application read only user, DB
     administrator etc.) and RabbitMQ doesn't need (shouldn't need) to know
     about them, you may want to create a filter for user's roles put into the
     JWT token via "Scope" tab where you select created `rabbitmq.*` roles
     (Filter by clients if you did not create roles as Realm roles).
6. Go back to Clients->Client scope tab and add client scope created above
   (with type set to `Default`).
7. Go to Groups and create some groups like
   ```
   webui-admins
   webui-mgmt
   ```
   In "Role mapping" tab of `webui-admins` group add
   ```
   rabbitmq.read:*/*/*
   rabbitmq.write:*/*/*
   rabbitmq.configure:*/*/*
   rabbitmq-web-uirabbitmq.tag:administrator
   ```
   (again Filter by clients if you did not create roles as Realm roles).
8. Go to Users and add a user and join the `webui-admins` group.
   View details of the created user and set a password in "Credentials" tab.

### RabbitMQ configuration

Using helm chart version 11.13.0 (RabbitMQ version 3.11.13 - OAuth2 is
supported since version 3.11) create following values.yaml file (just an
example for the testing):

```yaml
# The values below assume Bitnami upstream helm chart version 11.13.0 and
# RabbitMQ version 3.11.13.

auth:
  # Do not generate random password but use predefined one instead.
  existingPasswordSecret: rabbitmq-new-password

# Space separated list of URLs of plugins to download.
communityPlugins: "https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/download/3.11.1/rabbitmq_delayed_message_exchange-3.11.1.ez"
# Space separated list of downloaded plugins to enable.
extraPlugins: "rabbitmq_delayed_message_exchange"

# This is the configuration for OAuth2 using KeyCloak.
extraConfiguration: |
  # Primary authentication method.
  auth_backends.1 = rabbit_auth_backend_oauth2

  # User's roles are mapped to "roles" JWT field by KeyCloak mapper.
  auth_oauth2.additional_scopes_key = roles

  # KID in KeyCloak -> Realm settings -> <<realm>> -> Keys -> Keys list
  auth_oauth2.default_key = j11MTY..............................K1F8Rfc

  # Instead of getting user's name by default from "sub" field of JWT
  # token, try to get user's full name from "name" (see "full name"
  # KeyCloak mapper) field and if not available, try "preferred_username"
  # (a.k.a. login name).
  # This is just to show logged user's name instead of something like
  # d14a7626-b5e5-46d2-b11e-f099a85d7f1b.
  # Note: Non-intuitively the last one is used if found (bug?).
  auth_oauth2.preferred_username_claims.1 = preferred_username
  auth_oauth2.preferred_username_claims.2 = name

  # resource_server_id from KeyCloak configuration step 1.
  auth_oauth2.resource_server_id = rabbitmq
  # KID as index and for the certificate file see extraSecrets and extraVolumeMounts.
  auth_oauth2.signing_keys.j11MTY..............................K1F8Rfc = /bitnami/rabbitmq/conf/realm-certificate/realm-certificate.pem
  default_vhost = /

  log.connection.level = debug
  log.console.level = debug
  log.default.level = debug

  management.cors.allow_origins.1 = *
  # Allow only KeyCloak authentication method for accessing the web UI.
  management.disable_basic_auth = true
  # See KeyCloak configuration step 4.
  management.oauth_client_id = rabbitmq-web-ui
  # See KeyCloak configuration step 3.
  management.oauth_client_secret = Nei9....................UNI4SnAU
  management.oauth_enabled = true
  # The last URL part is the name of the KeyCloak realm.
  management.oauth_provider_url = https://keycloak.internal.example.com/realms/rabbitmq

extraSecrets:
  # See KeyCloak configuration step 3.
  # Certificate in KeyCloak -> Realm settings -> <<realm>> -> Keys -> Keys list
  realm-certificate:
    realm-certificate.pem: |
      -----BEGIN CERTIFICATE-----
      MIICnz..........................................................
      ................................................................
      ................................................................
      ................................................................
      ................................................................
      ................................................................
      ................................................................
      ................................................................
      ................................................................
      ................................................................
      ................................................................
      ................................................................
      ................................................................
      ................................................................
      UBTx
      -----END CERTIFICATE-----

extraSecretsPrependReleaseName: true

extraVolumes:
  - name: realm-certificate
    secret:
      # See extraSecretsPrependReleaseName: true
      secretName: rabbitmq-new-realm-certificate
      items:
      - key: realm-certificate.pem
        path: realm-certificate.pem

extraVolumeMounts:
  - name: realm-certificate
    mountPath: /bitnami/rabbitmq/conf/realm-certificate
    readOnly: true

# Adjust for your environment.
ingress:
  annotations:
    cert-manager.io/cluster-issuer: zerossl-production
    # Without this the communication between RabbitMQ and KeyCloak would not
    # work.
    nginx.ingress.kubernetes.io/enable-cors: "true"
  enabled: true
  hostname: rabbitmq-new.internal.example.com
  ingressClassName: ingress-internal
  tls: true

persistence:
  storageClass: gp3-er

# Space separated list of built-in plugins to enable.
plugins: >-
  rabbitmq_auth_backend_ldap
  rabbitmq_auth_backend_oauth2
  rabbitmq_management
  rabbitmq_peer_discovery_k8s
  rabbitmq_prometheus
  rabbitmq_shovel
  rabbitmq_shovel_management

resources:
  requests:
    memory: 256Mi
```
and run
```shell
helm install rabbitmq-new bitnami/rabbitmq --version 11.13.0 --values the_values_file_above.yaml
```
