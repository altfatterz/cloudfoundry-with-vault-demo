## Learning HashiCorp Vault

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
java -jar target/vault-demo-0.0.1-SNAPSHOT.jar --spring.cloud.vault.token=`echo $VAULT_TOKEN`
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

Password:s3cr3t
```

11. Send refresh command to the application

```bash
http post :8080/application/refresh
```

12. Verify that the application knows about the latest secret

```bash
http :8080

message:'Now, young Skywalker, you will die.'
```

## Running on CloudFoundry

```bash
git clone https://github.com/hashicorp/vault-service-broker
cd vault-service-broker

VAULT_ADDR=http://43cb69ee.ngrok.io
VAULT_TOKEN=d54a633e-e573-4ff1-adbe-7c9fe8180ab0
VAULT_USERNAME=vault
VAULT_PASSWORD=secret

cf push my-vault-broker-service -m 256M --random-route --no-start 

cf set-env my-vault-broker-service VAULT_ADDR ${VAULT_ADDR}
cf set-env my-vault-broker-service VAULT_TOKEN ${VAULT_TOKEN}
cf set-env my-vault-broker-service SECURITY_USER_NAME ${VAULT_USERNAME}
cf set-env my-vault-broker-service SECURITY_USER_PASSWORD ${VAULT_PASSWORD}

cf env my-vault-broker-service

```


```bash
cf start my-vault-broker-service
```

Check the Ngrok Inspect UI, you will see that 

```bash
POST /v1/sys/mounts/cf/broker
```
 
It created a new mount:
 
```
vault mounts

...
cf/broker/  generic    generic_4c6ea7ec    n/a     system       system   false           replicated
...
``` 

```
cf apps

name                      requested state   instances   memory   disk   urls
my-vault-broker-service   started           1/1         256M     1G     my-vault-broker-service-unlamed-nonporness.cfapps.io
``` 
 


```bash
VAULT_BROKER_URL=$(cf app my-vault-broker-service | grep routes: | awk '{print $2}')
```

Get the catalog information:

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

```bash
cf service-brokers
cf create-service-broker my-vault-service-broker "${VAULT_USERNAME}" "${VAULT_PASSWORD}" "https://${VAULT_BROKER_URL}" --space-scoped
```

You need to specify the `--space-scoped` and the `service ids` and `service name` must be unique. See `https://docs.cloudfoundry.org/services/managing-service-brokers.html`


```
cf marketplace
cf marketplace -s my-hashicorp-vault
``` 

Note the service name is `my-hashicorp-vault`.

Create a service instance:

```bash
cf create-service my-hashicorp-vault shared my-vault-service
``` 

Check result:

```bash
cf services

name               service              plan     bound apps   last operation
my-vault-service   my-hashicorp-vault   shared                create succeeded
```

Check the HTTP requests sent to Ngrok and you will see couple of vault mount being created:

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

Create a service key:

```bash
cf create-service-key my-vault-service my-vault-service-key
cf service-keys my-vault-service
```

This created more activity behind the scene

```bash
PUT  /v1/auth/token/renew-self                                                               200 OK
PUT  /v1/auth/token/renew-self                                                               200 OK
PUT  /v1/cf/broker/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/a4a878ba-60ef-4476-9862-b78b0bb514d3 204 No Content
POST /v1/auth/token/create/cf-41b2d6df-f7d1-453e-98e5-9a0bd1b2c347                           200 OK
```


Retrieve credentials for this instance:

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

```bash
cf push --random-route --no-start 
```


Inside vault the followings were created:

```bash
2017/11/06 21:53:39.209025 [INFO ] core: successful mount: path=cf/broker/ type=generic
```

```bash
2017/11/06 22:16:39.967360 [INFO ] core: successful mount: path=cf/9a491a25-b629-472a-bd65-137ce8067a8d/secret/ type=generic
2017/11/06 22:16:39.971579 [INFO ] core: successful mount: path=cf/5ba6ae64-640e-418b-8c0c-7ec4e9b5ace5/secret/ type=generic
2017/11/06 22:16:39.974906 [INFO ] core: successful mount: path=cf/ae0bc448-b371-4d00-8837-49fc4fc736c7/secret/ type=generic
2017/11/06 22:16:39.978739 [INFO ] core: successful mount: path=cf/ae0bc448-b371-4d00-8837-49fc4fc736c7/transit/ type=transit
```


```bash
Mount the generic backend at /cf/<organization_id>/secret/
Mount the generic backend at /cf/<space_id>/secret/
Mount the generic backend at /cf/<instance_id>/secret/
Mount the transit backend at /cf/<instance_id>/transit/
```
 
```bash
vault write cf/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/secret/my-vault-demo password=Work 
http post http://my-vault-demo-deleterious-geum.cfapps.io/application/refresh
``` 
 
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