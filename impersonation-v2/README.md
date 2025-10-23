# this example shows impersonation based on admin-fine-grained-authz:v2

## run keycloak

### in docker

```
git clone https://github.com/flyinfish/keycloak-examples.git
cd keycloak-examples/impersonation-v2
docker-compose up
```

login with master-admin (admin:admin) http://localhost:8883/admin/master/console/#/dev/users.

all users (except `test*` and `c*`) have a password matching their username, this means that `test*` and `c*` cannot login, they can only be impersonated.

all users have readonly-access to realm `dev` via composite-role [dev-ro](http://localhost:8883/admin/dev/console/#/dev/roles)

## scenarios

### developer `dev01` is allowed to impersonate testusers `test01`

go to [users](http://localhost:8883/admin/dev/console/#/dev/users) and login as `dev01`. <br>
select any user you wish, only [test01](http://localhost:8883/admin/dev/console/#/dev/users/275bdbd8-2c5b-4d1a-b838-a7997ed3b85e/settings) can be impersonated top right in "Action"-dropdown.

#### this is how it works

there is a group [testusers](http://localhost:8883/admin/dev/console/#/dev/groups) with `test01` as member.<br>
the group-permission [testusers can be impersonated by users in role developer](http://localhost:8883/admin/dev/console/#/dev/permissions) for group `/testusers` with Authorization Scope `impersonate-members` is granted by policy [in-role-developer](http://localhost:8883/admin/dev/console/#/dev/permissions/4a32fd7a-05f3-49b7-94db-676ede8457e5/policies) to all users with role `developer`

### customer `m01` can impersonate to its profiles `c11`,`c12`

go to [users](http://localhost:8883/admin/dev/console/#/dev/users) and login as `m01` - you cannot login as `c11` or `c12` since these have no credentials, you only can impersonate to these users. <br>
select any user you wish, only [c11 or c12](http://localhost:8883/admin/dev/console/#/dev/users/275bdbd8-2c5b-4d1a-b838-a7997ed3b85e/settings) can be impersonated top right in "Action"-dropdown.

#### this is how it works

there is a group [m01-impersonation-group](http://localhost:8883/admin/dev/console/#/dev/groups) with `m01` `c11` `c12` as members.<br>
the group-permission [m01-customers-can-impersonate-each-other](http://localhost:8883/admin/dev/console/#/dev/permissions) for group `/m01-impersonation-group` with Authorization Scope `impersonate-members` is granted by policy [m01-impersonation-group-policy](http://localhost:8883/admin/dev/console/#/dev/permissions/4a32fd7a-05f3-49b7-94db-676ede8457e5/policies) to all users from the same group.

#### !!however!!

while this works  perfectly for a single group of one customer `m01` and its profiles `c11`, `c12` this will not scale for (ten|hundred) tousands of customers.




## export realm

```
container="keycloak-examples-impersonation-v2" \
&& docker exec -it $container sh -c \
  "cp -rp /opt/keycloak/data/h2 /tmp ; \
  /opt/keycloak/bin/kc.sh export --realm dev --file /import/dev-realm.json \
    --db dev-file \
    --db-url 'jdbc:h2:file:/tmp/h2/keycloakdb;NON_KEYWORDS=VALUE'"\
&& docker cp $container:/tmp/dev-realm.json  impersonation-v2/import/dev-realm.json
```

