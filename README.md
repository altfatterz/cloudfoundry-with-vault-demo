## Using HashiCorp Vault in CloudFoundry  

#### Running locally

1. Start vault:

```bash
vault server -config inmemory.conf
```

2. In another terminal set the VAULT_ADDR before initializing Vault:

```bash
export VAULT_ADDR=http://127.0.0.1:8200
```   

2. Initialize vault:

```bash
vault init -key-shares=5 -key-threshold=2
```

3. Copy the `Initial Root Token` we will still need it.

```bash
export VAULT_TOKEN=<token>
```

Vault requires an authenticated access to proceed from here on. 
Vault uses tokens as generic authentication on its transport level.

4. Vault is in `sealed` mode, let's unseal it:

```
vault unseal <key>
vault unseal <key>
```

5. Verify that Vault is in `unsealed` mode:

```bash
vault status | grep Sealed

Sealed: false
```

6. Write a secret into the `secret` backend:

```bash
vault write secret/vault-demo message='I find your lack of faith disturbing.'
```

7. Start the application in another terminal 

```bash
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=<token>
java -jar target/cloudfoundry-with-vault-demo-0.0.1-SNAPSHOT.jar --spring.cloud.vault.token=`echo $VAULT_TOKEN`
```

8. Request the GET localhost:8080

```bash
http :8080

message:I find your lack of faith disturbing.
```

9. Update the secret inside Vault

```bash
vault write secret/vault-demo message='Now, young Skywalker, you will die.'
```

10. Verify that the application still has the old secret

```bash
http :8080

message:I find your lack of faith disturbing.
```

11. Send refresh command to the application

```bash
http post :8080/actuator/refresh
```

12. Verify that the application knows about the latest secret

```bash
http :8080

message:'Now, young Skywalker, you will die.'
```

## Running on CloudFoundry

1. Expose the locally running vault via [ngrok](https://ngrok.com/)

```
ngrok http 8200

Forwarding http://3db1eef2.ngrok.io -> localhost:8200
Forwarding https://3db1eef2.ngrok.io -> localhost:8200
```

2. Verify on the web interface at `http://localhost:4040`

3. Get the [Open Service Broker API](https://www.openservicebrokerapi.org/) implementation from HashiCorp:

```bash
git clone https://github.com/hashicorp/vault-service-broker
```

3. Needed to change the `DefaultServiceID` and `DefaultServiceName` in the `main.go` file

4. Set the following environment variables

```bash
VAULT_ADDR=<ngrok_url>
VAULT_TOKEN=<token>
```

The broker is configured to use basic authentication

```bash
VAULT_USERNAME=vault
VAULT_PASSWORD=secret
```

5. Deploy the broker to CloudFoundry

```bash
cf push my-vault-broker-service -m 256M --random-route --no-start 
```

The `--no-start` makes sure it is not started after it is deployed.


6. Configure the environment variables

```bash
cf set-env my-vault-broker-service VAULT_ADDR ${VAULT_ADDR}
cf set-env my-vault-broker-service VAULT_TOKEN ${VAULT_TOKEN}
cf set-env my-vault-broker-service SECURITY_USER_NAME ${VAULT_USERNAME}
cf set-env my-vault-broker-service SECURITY_USER_PASSWORD ${VAULT_PASSWORD}
```

7. Verify the configured environment variables

```bash
cf env my-vault-broker-service
```

8. Start the broker:

```bash
cf start my-vault-broker-service
```

9. Check the Ngrok Inspect UI, you will see the following request: 

```bash
POST /v1/sys/mounts/cf/broker
```
 
it created a new mount:
 
```
vault mounts

...
cf/broker/  generic    generic_4c6ea7ec    n/a     system       system   false           replicated
...
``` 


10. View the running broker:

```
cf apps

name                      requested state   instances   memory   disk   urls
my-vault-broker-service   started           1/1         256M     1G     my-vault-broker-service-<name>.cfapps.io
``` 
 

11. Get the broker url:

```bash
VAULT_BROKER_URL=$(cf app my-vault-broker-service | grep routes: | awk '{print $2}')
```

12. Get the catalog information:

```bash
curl ${VAULT_USERNAME}:${VAULT_PASSWORD}@${VAULT_BROKER_URL}/v2/catalog | jq
```

```json
{
  "services": [
    {
      "id": "42ff1ff1-244d-413a-87ab-b2334b801134",
      "name": "my-hashicorp-vault",
      "description": "HashiCorp Vault Service Broker",
      "bindable": true,
      "tags": [
        ""
      ],
      "plan_updateable": false,
      "plans": [
        {
          "id": "42ff1ff1-244d-413a-87ab-b2334b801134.shared",
          "name": "shared",
          "description": "Secure access to Vault's storage and transit backends",
          "free": true
        }
      ]
    }
  ]
}
```

13. Create a service broker:

```bash
cf service-brokers
cf create-service-broker my-vault-service-broker "${VAULT_USERNAME}" "${VAULT_PASSWORD}" "https://${VAULT_BROKER_URL}" --space-scoped
```

You need to specify the `--space-scoped` and the `service ids` and `service name` must be unique. See `https://docs.cloudfoundry.org/services/managing-service-brokers.html`

14. Create a service instance:

```bash
cf create-service my-hashicorp-vault shared my-vault-service
``` 

15. Verify the result:

```bash
cf services

name               service              plan     bound apps   last operation
my-vault-service   my-hashicorp-vault   shared                create succeeded
```

16. Check the HTTP requests sent to Ngrok and you will see couple of vault mounts being created:

```bash
PUT /v1/cf/broker/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347              204 No Content
POST /v1/sys/mounts/cf/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/transit 204 No Content
POST /v1/sys/mounts/cf/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/secret  204 No Content
POST /v1/sys/mounts/cf/5f7b0811-d90a-47f2-a194-951eb324f867/secret  204 No Content
POST /v1/sys/mounts/cf/be7eedf8-c813-49e1-98f8-2fc19370ee4d/secret  204 No Content
GET /v1/sys/mounts                                                  200 Ok
PUT /v1/auth/token/roles/cf-41b2d6df-f7d1-453e-98e5-9a0bd1b2c347    204 No Content
PUT /v1/sys/policy/cf-41b2d6df-f7d1-453e-98e5-9a0bd1b2c347          204 No Content	
```

When  a new service instance is provisioned using the broker, the following paths will be mounted:

Mount the generic backend at /cf/<organization_id>/secret/
Mount the generic backend at /cf/<space_id>/secret/
Mount the generic backend at /cf/<instance_id>/secret/
Mount the transit backend at /cf/<instance_id>/transit/

A policy named `cf-<instance_id>` is also created for this service instance which grants read-only access to `cf/<organization_id>/*`, read-write access to `cf/<space_id>/*` and full access to `cf/<instance_id>/*`

17. Create a service key:

```bash
cf create-service-key my-vault-service my-vault-service-key
cf service-keys my-vault-service
```

18. Verify the received requests for Vault using the Ngrok Inspect UI

```bash
PUT  /v1/auth/token/renew-self                                                               200 OK
PUT  /v1/auth/token/renew-self                                                               200 OK
PUT  /v1/cf/broker/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/a4a878ba-60ef-4476-9862-b78b0bb514d3 204 No Content
POST /v1/auth/token/create/cf-41b2d6df-f7d1-453e-98e5-9a0bd1b2c347                           200 OK
```

19. Retrieve credentials for this instance:

```bash
cf service-key my-vault-service my-vault-service-key
```

```json
{
 "address": "http://43cb69ee.ngrok.io/",
 "auth": {
  "accessor": "d994b172-c116-1d56-39c4-139f9615abb4",
  "token": "e340fdbe-373a-0b24-0a9c-806316b379a6"
 },
 "backends": {
  "generic": "cf/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/secret",
  "transit": "cf/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/transit"
 },
 "backends_shared": {
  "organization": "cf/be7eedf8-c813-49e1-98f8-2fc19370ee4d/secret",
  "space": "cf/5f7b0811-d90a-47f2-a194-951eb324f867/secret"
 }
}
```

In the application, we can leverage these services using the following configuration in the bootstrap.yml file. Note that we are only able to access the exposed backends.
`
```bash
spring:
  application:
    name: vault-demo
  cloud:
    vault:
      token: ${vcap.services.my-vault-service.credentials.auth.token}
      uri: ${vcap.services.my-vault-service.credentials.address:http://localhost:8200}
      generic:
        backend: ${vcap.services.my-vault-service.credentials.backends.generic:secret}
```

After redeploying with the above changes, let's write the secret into the vault to the given generic backend.

```bash
vault write cf/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/secret/vault-demo message='Vault Rocks'
http post http://vault-demo-deleterious-geum.cfapps.io/application/refresh
```

We can verify that the secret is retrieved via

```bash
http get http://vault-demo-deleterious-geum.cfapps.io
```

--


How can I connect apps running in PCF Dev to services running on my workstation?

Note: Using localhost inside of app containers will not refer to your workstation.

PCF Dev provides a special hostname for addressing the host from inside of application containers. 
If the PCF Dev system domain is local.pcfdev.io, then the host will be routable at host.pcfdev.io.
Services running on the host must be listening on all network interfaces (not just localhost) for apps to access them.

 
Look into:
Spring Cloud project for creating Cloud Foundry service brokers
`https://github.com/spring-cloud/spring-cloud-cloudfoundry-service-broker`

 
 
Resources:
[https://spring.io/blog/2016/06/24/managing-secrets-with-vault](https://spring.io/blog/2016/06/24/managing-secrets-with-vault)
[https://spring.io/blog/2015/04/27/binding-to-data-services-with-spring-boot-in-cloud-foundry](https://spring.io/blog/2015/04/27/binding-to-data-services-with-spring-boot-in-cloud-foundry)