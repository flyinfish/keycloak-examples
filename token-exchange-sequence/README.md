# this example shows token exchange sequence

Initially only [S1](#s1-token-exchange-starting-with-grant_typepassword-on-service-1-client) worked
https://stackoverflow.com/questions/78602573/keycloak-token-exchange-sequence

Then issue https://github.com/keycloak/keycloak/issues/30614 fixed [S2](#s2-token-exchange-starting-with-grant_typeclient_credentials-on-service-1-client)

[S3](#s3-token-exchange-starting-with-grant_typepassword-on-public-spa) still fails

## run keycloak

### in docker

```
cd token-exchange-sequence
docker-compose up
```

### form your locally built keycloak

first build it (assuming keycloak cloned in same root as this repo)

```
cd keycloak
./mvnw -DskipTests clean install
java -Dkc.home.dir=./quarkus/server/target/ -jar quarkus/server/target/lib/quarkus-run.jar build --features=preview
java -Dkc.home.dir=./quarkus/server/target/ -jar quarkus/server/target/lib/quarkus-run.jar show-config
java -Dkc.home.dir=./quarkus/server/target/ -jar quarkus/server/target/lib/quarkus-run.jar import --file ../keycloak-examples/token-exchange-sequence/import/dev-realm.json
```

then run it

```
KEYCLOAK_ADMIN=admin 
KEYCLOAK_ADMIN_PASSWORD=admin 

java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8787 -Dkc.home.dir=./quarkus/server/target/ -jar quarkus/server/target/lib/quarkus-run.jar start-dev --http-port 8881 --log-level=INFO,org.keycloak.services.managers:debug,org.keycloak.authentication:debug,org.keycloak.events:debug
```

## scenarios

there are 3 clients `service-1-client`, `service-2-client`, `service-3-client` and `public-spa`.

### S1: token-exchange starting with grant_type=password on `service-1-client`

Works fine with quay.io/keycloak/keycloak:24.0.5

1. get a token for `service-1-client` using client-credentials and password
2. then exchange token for audience `service-2-client` using client `service-1-client`
3. then exchange token for audience `service-3-client` using client `service-2-client`

```
  # 1
  token1=$(curl -s -X POST \
    --location http://localhost:8881/realms/dev/protocol/openid-connect/token \
    --header "Content-Type: application/x-www-form-urlencoded" \
    --data-urlencode "client_id=service-1-client" \
    --data-urlencode "client_secret=LgxGBWTyGvL1YqpeMPcUPprZpXUv34MR" \
    --data-urlencode "grant_type=password" \
    --data-urlencode "username=grant" \
    --data-urlencode "password=grant" \
    | jq -r '.access_token')
  echo $token1  

  # 2
  token2=$(curl -s -X POST \
    --location http://localhost:8881/realms/dev/protocol/openid-connect/token \
    --header "Content-Type: application/x-www-form-urlencoded" \
    --data-urlencode "client_id=service-1-client" \
    --data-urlencode "client_secret=LgxGBWTyGvL1YqpeMPcUPprZpXUv34MR" \
    --data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
    --data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
    --data-urlencode "audience=service-2-client" \
    --data-urlencode "subject_token=$token1" | jq -r '.access_token')
  echo $token2  

  # 3
  curl -s -i -X POST \
    --location http://localhost:8881/realms/dev/protocol/openid-connect/token \
    --header "Content-Type: application/x-www-form-urlencoded" \
    --data-urlencode "client_id=service-2-client" \
    --data-urlencode "client_secret=UjGgHI0WsypbrtZtr2bTTbjKJll8TeDO" \
    --data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
    --data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
    --data-urlencode "audience=service-3-client" \
    --data-urlencode "subject_token=$token2"
    
``` 

### S2: token-exchange starting with grant_type=client_credentials on `service-1-client`

1. get a token for `service-1-client` using client credentials
2. exchange token for audience `service-2-client` using client `service-1-client`
3. exchange token for audience `service-3-client` using client `service-2-client`

works like a charm - was fixed by https://github.com/keycloak/keycloak/pull/30975/files
trough https://github.com/keycloak/keycloak/issues/30614

```
  # 1
  token1=$(curl -s -X POST \
    --location http://localhost:8881/realms/dev/protocol/openid-connect/token \
    --header "Content-Type: application/x-www-form-urlencoded" \
    --data-urlencode "client_id=service-1-client" \
    --data-urlencode "client_secret=LgxGBWTyGvL1YqpeMPcUPprZpXUv34MR" \
    --data-urlencode "grant_type=client_credentials" | jq -r '.access_token')
  echo $token1  

  # 2
  token2=$(curl -s -X POST \
    --location http://localhost:8881/realms/dev/protocol/openid-connect/token \
    --header "Content-Type: application/x-www-form-urlencoded" \
    --data-urlencode "client_id=service-1-client" \
    --data-urlencode "client_secret=LgxGBWTyGvL1YqpeMPcUPprZpXUv34MR" \
    --data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
    --data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
    --data-urlencode "audience=service-2-client" \
    --data-urlencode "subject_token=$token1" | jq -r '.access_token')
  echo $token2  

  # 3
  curl -s -i -X POST \
    --location http://localhost:8881/realms/dev/protocol/openid-connect/token \
    --header "Content-Type: application/x-www-form-urlencoded" \
    --data-urlencode "client_id=service-2-client" \
    --data-urlencode "client_secret=UjGgHI0WsypbrtZtr2bTTbjKJll8TeDO" \
    --data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
    --data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
    --data-urlencode "audience=service-3-client" \
    --data-urlencode "subject_token=$token2"    
``` 

### S3: token-exchange starting with grant_type=password on `public-spa`

Still fails

1. get a token for `public-spa` using password.
2. exchange token for audience `service-2-client` using client `service-1-client`
3. exchange token for audience `service-3-client` using client `service-2-client`

unfortunately this step 3 fails with `Http 400 Bad Request - invalid_token`
logging

```
DEBUG [org.keycloak.services.managers.AuthenticationManager] (executor-thread-25) Client session for client 'service-1-client' not present in user session '...'
```

```
  # 1
  token1=$(curl -s -X POST \
    --location http://localhost:8881/realms/dev/protocol/openid-connect/token \
    --header "Content-Type: application/x-www-form-urlencoded" \
    --data-urlencode "client_id=public-spa" \
    --data-urlencode "grant_type=password" \
    --data-urlencode "username=grant" \
    --data-urlencode "password=grant" \
    | jq -r '.access_token')
  echo $token1  
    
  # 2
  token2=$(curl -s -X POST \
    --location http://localhost:8881/realms/dev/protocol/openid-connect/token \
    --header "Content-Type: application/x-www-form-urlencoded" \
    --data-urlencode "client_id=service-1-client" \
    --data-urlencode "client_secret=LgxGBWTyGvL1YqpeMPcUPprZpXUv34MR" \
    --data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
    --data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
    --data-urlencode "audience=service-2-client" \
    --data-urlencode "subject_token=$token1" | jq -r '.access_token')
  echo $token2  

  # 3
  curl -s -i -X POST \
    --location http://localhost:8881/realms/dev/protocol/openid-connect/token \
    --header "Content-Type: application/x-www-form-urlencoded" \
    --data-urlencode "client_id=service-2-client" \
    --data-urlencode "client_secret=UjGgHI0WsypbrtZtr2bTTbjKJll8TeDO" \
    --data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
    --data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
    --data-urlencode "audience=service-3-client" \
    --data-urlencode "subject_token=$token2"
    
``` 


## export realm

```
container="keycloak-examples-multi-service-token-exchange" \
&& docker exec $container /opt/keycloak/bin/kc.sh export --realm dev --file /tmp/dev-realm.json \
&& docker cp $container:/tmp/dev-realm.json  token-exchange-sequence/import/dev-realm.json
```

