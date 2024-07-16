# this example shows impersonation


## run keycloak

### in docker

```
cd token-exchange-sequence
docker-compose up
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
* `impersonation-admin` must be allowed to exchange token for `public-spa` ([permissions/token-exchange](http://localhost:8882/admin/master/console/#/dev/clients/2575df73-bf3d-458f-b5f3-8eaacf134c39/permissions))
* `grant` should must be allowed to impersonate [users/permissions/impersonate](http://localhost:8882/admin/master/console/#/dev/users/permissions)
<br> out of the box this might be achieved by adding role `impersonation` to grant.
* if there are any policies defined on [users/permissions/user-impersonated](http://localhost:8882/admin/master/console/#/dev/users/permissions) there must allow too.




## export realm

```
container="keycloak-examples-impersonation" \
&& docker exec -e DEBUG_PORT="*:8788" $container /opt/keycloak/bin/kc.sh export --realm dev --file /tmp/dev-realm.json --http-management-port 9001 --debug \
&& docker cp $container:/tmp/dev-realm.json  impersonation/import/dev-realm.json
```

