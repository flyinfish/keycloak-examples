# token exchange sequence

## run keycloak
```
cd token-exchange-sequence
docker-compose up
```

## scenarios
there are 3 clients `service-1-client`, `service-2-client`, `service-3-client` and `public-spa`.

### token-exchange starting with grant_type=client_credentials on `service-1-client`
1. get a token for `service-1-client` using client credentials<
2. exchange token for audience `service-2-client` using client `service-1-client`
3. exchange token for audience `service-3-client` using client `service-2-client`

unfortunately this step 3 fails with `Http 400 Bad Request - invalid_token`
logging
```
DEBUG [org.keycloak.services.managers.AuthenticationManager] (executor-thread-25) Client session for client 'service-1-client' not present in user session 'bcc55f7b-bf6a-488b-99af-f46cf4e8ef6c'
```

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

### token-exchange starting with grant_type=password on `public-spa`
1. get a token for `public-spa` using password.
2. exchange token for audience `service-2-client` using client `service-1-client`
3. exchange token for audience `service-3-client` using client `service-2-client`

unfortunately this step 3 fails with `Http 400 Bad Request - invalid_token`
logging
```
DEBUG [org.keycloak.services.managers.AuthenticationManager] (executor-thread-25) Client session for client 'service-1-client' not present in user session 'bcc55f7b-bf6a-488b-99af-f46cf4e8ef6c'
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

### token-exchange starting with grant_type=password on `service-1-client`
1. get a token for `service-1-client` using client-credentials<br>
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

## export realm
```
container="keycloak-examples-multi-service-token-exchange" \
&& docker exec $container /opt/keycloak/bin/kc.sh export --realm dev --file /tmp/dev-realm.json \
&& docker cp $container:/tmp/dev-realm.json  token-exchange-sequence/import/dev-realm.json
```

