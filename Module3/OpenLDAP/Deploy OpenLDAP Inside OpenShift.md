## Deploy OpenLDAP Inside OpenShift
#### Create Project
```sh
oc new-project ldap-demo
```
#### Deploy OpenLDAP
```sh
oc create -f openldap.yaml
oc create -f service.yaml

#Load Users into LDAP
oc rsh deploy/openldap

#Run:
dsconf localhost backend create \
  --suffix dc=acme,dc=com \
  --be-name userRoot \
  --create-suffix
```
#### Verify:
```sh
dsconf localhost backend suffix list

#Expected:
#dc=acme,dc=com
```
### Then verify the suffix exists:
```sh
ldapsearch \
-x \
-H ldap://localhost:3389 \
-D "cn=Directory Manager" \
-w RedHat123! \
-b "dc=acme,dc=com"
```

#### You should now see something similar to:
```sh
dn: dc=acme,dc=com
objectClass: top
objectClass: domain
dc: acme
```

#### Create Users OU
```sh
cat >/tmp/ou.ldif <<'EOF'
dn: ou=users,dc=acme,dc=com
objectClass: organizationalUnit
ou: users
EOF
```
### Import:
```sh
ldapadd \
-x \
-H ldap://localhost:3389 \
-D "cn=Directory Manager" \
-w RedHat123! \
-f /tmp/ou.ldif
```

#### Create Users
```sh
cat >/tmp/users.ldif <<'EOF'
dn: uid=john,ou=users,dc=acme,dc=com
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
cn: John Doe
sn: Doe
uid: john
mail: john@acme.com
userPassword: Password123

dn: uid=alice,ou=users,dc=acme,dc=com
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
cn: Alice Smith
sn: Smith
uid: alice
mail: alice@acme.com
userPassword: Password123

dn: uid=dev1,ou=users,dc=acme,dc=com
objectClass: inetOrgPerson
cn: Dev One
sn: One
uid: dev1
mail: dev1@acme.com
userPassword: Password123

dn: uid=dev2,ou=users,dc=acme,dc=com
objectClass: inetOrgPerson
cn: Dev Two
sn: Two
uid: dev2
mail: dev2@acme.com
userPassword: Password123

dn: uid=dev3,ou=users,dc=acme,dc=com
objectClass: inetOrgPerson
cn: Dev Three
sn: Three
uid: dev3
mail: dev3@acme.com
userPassword: Password123

dn: uid=dev4,ou=users,dc=acme,dc=com
objectClass: inetOrgPerson
cn: Dev Four
sn: Four
uid: dev4
mail: dev4@acme.com
userPassword: Password123
EOF
```
#### Import:
```sh
ldapadd \
-x \
-H ldap://localhost:3389 \
-D "cn=Directory Manager" \
-w RedHat123! \
-f /tmp/users.ldif
```

#### Validate Authentication

#### Search for users:
```sh
ldapsearch \
-x \
-H ldap://localhost:3389 \
-D "cn=Directory Manager" \
-w RedHat123! \
-b "ou=users,dc=acme,dc=com"
```
#### Test a user bind:
```sh
ldapwhoami \
-x \
-H ldap://localhost:3389 \
-D "uid=john,ou=users,dc=acme,dc=com" \
-w Password123
```

#### Expected:
```sh
dn:uid=john,ou=users,dc=acme,dc=com
```

#### Create Bind Account for OpenShift
```sh
cat >/tmp/bind.ldif <<'EOF'
dn: ou=serviceaccounts,dc=acme,dc=com
objectClass: organizationalUnit
ou: serviceaccounts

dn: cn=ldapbind,ou=serviceaccounts,dc=acme,dc=com
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: ldapbind
userPassword: BindPassword123
EOF
```
#### Import:
```sh
ldapadd \
-x \
-H ldap://localhost:3389 \
-D "cn=Directory Manager" \
-w RedHat123! \
-f /tmp/bind.ldif
```

#### Verify:
```sh
ldapsearch \
-x \
-H ldap://localhost:3389 \
-D "cn=Directory Manager" \
-w RedHat123! \
-b "ou=serviceaccounts,dc=acme,dc=com"
```

#### Before Configuring OpenShift OAuth

#### Verify these four commands succeed:
```sh
dsconf localhost backend suffix list

ldapsearch -x -H ldap://localhost:3389 \
-D "cn=Directory Manager" \
-w RedHat123! \
-b "dc=acme,dc=com"

ldapsearch -x -H ldap://localhost:3389 \
-D "cn=Directory Manager" \
-w RedHat123! \
-b "ou=users,dc=acme,dc=com"

ldapwhoami -x -H ldap://localhost:3389 \
-D "uid=john,ou=users,dc=acme,dc=com" \
-w Password123
```
Once all four work, you can move on to the OpenShift side and configure the OAuth LDAP identity provider correctly for your SNO cluster.

```sh
oc create secret generic ldap-secret \
  --from-literal=bindPassword='RedHat123!' \
  -n openshift-config
```

##### Troubelshooting
```sh
ldapsearch -x -H ldap://ds389.ldap-demo.svc.cluster.local:389 -D "cn=ldapbind,ou=serviceaccounts,dc=acme,dc=com" -w BindPassword123 -b "ou=users,dc=acme,dc=com"

ldapsearch \
-x \
-H ldap://ds389.ldap-demo.svc.cluster.local:389 \
-D "cn=Directory Manager" \
-w RedHat123! \
-b "dc=acme,dc=com"
```