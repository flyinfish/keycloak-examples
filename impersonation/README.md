# this example shows impersonation

## run keycloak

### in docker

```
cd impersonation
docker-compose up
```

### form your locally built keycloak

first build it (assuming keycloak cloned in same root as this repo)

```
cd ../keycloak
./mvnw -DskipTests clean install
java -Dkc.home.dir=./quarkus/server/target/ -jar quarkus/server/target/lib/quarkus-run.jar build --features=preview
java -Dkc.home.dir=./quarkus/server/target/ -jar quarkus/server/target/lib/quarkus-run.jar show-config
java -Dkc.home.dir=./quarkus/server/target/ -jar quarkus/server/target/lib/quarkus-run.jar import --file ../keycloak-examples/impersonation/import/dev-realm.json
```

then run it

```
KEYCLOAK_ADMIN=admin 
KEYCLOAK_ADMIN_PASSWORD=admin 

java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8787 -Dkc.home.dir=./quarkus/server/target/ -jar quarkus/server/target/lib/quarkus-run.jar start-dev --http-port 8882 --log-level=INFO,org.keycloak.authorization.policy.provider:debug
```

## scenarios

### S1 - impersonation via tokenexchange

user `grant` logs in with client-id `public-spa`. since he is a tester he wants to impersonate `testuser001`.
for this he uses the app/client-id `impersonation-admin` providing him an impersonated token.

[Securing Apps / 7.5. Impersonation](https://www.keycloak.org/docs/latest/securing_apps/#impersonation)

```
token1=$(curl -s -X POST \
--location http://localhost:8882/realms/dev/protocol/openid-connect/token \
--header "Content-Type: application/x-www-form-urlencoded" \
--data-urlencode "client_id=public-spa" \
--data-urlencode "grant_type=password" \
--data-urlencode "username=grant" \
--data-urlencode "password=grant" \
| jq -r '.access_token')
echo $token1  

curl -s -X POST \
--location http://localhost:8882/realms/dev/protocol/openid-connect/token \
--header "Content-Type: application/x-www-form-urlencoded" \
--data-urlencode "client_id=impersonation-admin" \
--data-urlencode "client_secret=xKeD7tNA6WJHSw1tbMoQKhxfpVyXdFts" \
--data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
--data-urlencode "subject_token=$token1" \
--data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
--data-urlencode "audience=public-spa" \
--data-urlencode "requested_subject=testuser001" | jq
```

#### required config

* client `impersonation-admin` must be allowed to exchange token
  for `public-spa` ([permissions/`token-exchange`](http://localhost:8882/admin/master/console/#/dev/clients/2575df73-bf3d-458f-b5f3-8eaacf134c39/permissions))
* `grant` should must be allowed to
  impersonate [users/permissions/`impersonate`](http://localhost:8882/admin/master/console/#/dev/users/permissions)
  <br> out of the box this might be achieved by adding role `impersonation` to grant.
* if there are any policies defined
  on [users/permissions/`user-impersonated`](http://localhost:8882/admin/master/console/#/dev/users/permissions) there must allow
  too.

### S2 - direct naked impersonation

for whatever reason we have a client `impersonation-direct-admin` which is allowed to impersonate without providing a user-token.
[Securing Apps / 7.6. Direct Naked Impersonation](https://www.keycloak.org/docs/latest/securing_apps/index.html#direct-naked-impersonation)

```
curl -s -X POST \
--location http://localhost:8882/realms/dev/protocol/openid-connect/token \
--header "Content-Type: application/x-www-form-urlencoded" \
--data-urlencode "client_id=impersonation-direct-admin" \
--data-urlencode "client_secret=r7cTLqAFCfBluWUCuhVFdd3S2jPOK474" \
--data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
--data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
--data-urlencode "requested_subject=grant" | jq

curl -s -X POST \
--location http://localhost:8882/realms/dev/protocol/openid-connect/token \
--header "Content-Type: application/x-www-form-urlencoded" \
--data-urlencode "client_id=impersonation-direct-admin" \
--data-urlencode "client_secret=r7cTLqAFCfBluWUCuhVFdd3S2jPOK474" \
--data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
--data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
--data-urlencode "audience=public-spa" \
--data-urlencode "requested_subject=grant" | jq
```

#### required config

* client `impersonation-direct-admin` must explicitly be allowed
  in [users/permissions/`impersonate`](http://localhost:8882/admin/master/console/#/dev/users/permissions)
* when requesting token for specific audience `--data-urlencode "audience=public-spa"` this client must allow it
  in [permissions/`token-exchange`](http://localhost:8882/admin/master/console/#/dev/clients/2575df73-bf3d-458f-b5f3-8eaacf134c39/permissions)

### S3 - conditional impersonation

https://github.com/keycloak/keycloak/discussions/20252

now its getting trickier. what if when we want to allow certain rules on `users-impersonated` for different `impersonate`.
how ca this be tied together? wouln't we need rules on `impersonate-&&-users-impersonated` which let us access.

#### S3.1 users with role tester should be able to impersonate only users starting with testuser

this can easily be solved by attaching a regex-policy
to [users/permissions/`users-impersonated`](http://localhost:8882/admin/master/console/#/dev/users/permissions) (not done yet in
this example: attach policy `users-starting-with-testuser` manually) if you like to check.
but then this policy is applied for all `impersonate`
and [S3.2](#s32-impersonation-direct-admin-should-still-be-able-to-impersonate-all-users) does not work anymore

```
token1=$(curl -s -X POST \
--location http://localhost:8882/realms/dev/protocol/openid-connect/token \
--header "Content-Type: application/x-www-form-urlencoded" \
--data-urlencode "client_id=public-spa" \
--data-urlencode "grant_type=password" \
--data-urlencode "username=grant" \
--data-urlencode "password=grant" \
| jq -r '.access_token')
 echo $token1  

 # should work
curl -s -X POST \
--location http://localhost:8882/realms/dev/protocol/openid-connect/token \
--header "Content-Type: application/x-www-form-urlencoded" \
--data-urlencode "client_id=impersonation-admin" \
--data-urlencode "client_secret=xKeD7tNA6WJHSw1tbMoQKhxfpVyXdFts" \
--data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
--data-urlencode "subject_token=$token1" \
--data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
--data-urlencode "audience=public-spa" \
--data-urlencode "requested_subject=testuser001" | jq

 # should fail
curl -s -X POST \
--location http://localhost:8882/realms/dev/protocol/openid-connect/token \
--header "Content-Type: application/x-www-form-urlencoded" \
--data-urlencode "client_id=impersonation-admin" \
--data-urlencode "client_secret=xKeD7tNA6WJHSw1tbMoQKhxfpVyXdFts" \
--data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
--data-urlencode "subject_token=$token1" \
--data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
--data-urlencode "audience=public-spa" \
--data-urlencode "requested_subject=devadmin" | jq
```

#### S3.2 `impersonation-direct-admin` should still be able to impersonate ALL users

as explaine above, shouldnt [UserPermissions](https://github.com/keycloak/keycloak/blob/35b9d8aa496bfe827e32adbd48e5eabb4ab182e7/services/src/main/java/org/keycloak/services/resources/admin/permissions/UserPermissions.java#L355)
```
return canImpersonate(context) && isImpersonatable(user) && isImpersonatableWithinContext(context, user);
```
rather than just
```
return canImpersonate(context) && isImpersonatable(user);
```


```
 # does still work
curl -s -X POST \
--location http://localhost:8882/realms/dev/protocol/openid-connect/token \
--header "Content-Type: application/x-www-form-urlencoded" \
--data-urlencode "client_id=impersonation-direct-admin" \
--data-urlencode "client_secret=r7cTLqAFCfBluWUCuhVFdd3S2jPOK474" \
--data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
--data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
--data-urlencode "audience=public-spa" \
--data-urlencode "requested_subject=testuser001" | jq

 # should work, but dont when compromised by S3.1 with policy `users-starting-with-testuser` attached
curl -s -X POST \
--location http://localhost:8882/realms/dev/protocol/openid-connect/token \
--header "Content-Type: application/x-www-form-urlencoded" \
--data-urlencode "client_id=impersonation-direct-admin" \
--data-urlencode "client_secret=r7cTLqAFCfBluWUCuhVFdd3S2jPOK474" \
--data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
--data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
--data-urlencode "requested_subject=grant" | jq
```

## export realm

```
container="keycloak-examples-impersonation" \
&& docker exec -e DEBUG_PORT="*:8788" $container /opt/keycloak/bin/kc.sh export --realm dev --file /tmp/dev-realm.json --http-management-port 9001 --debug \
&& docker cp $container:/tmp/dev-realm.json  impersonation/import/dev-realm.json
```

