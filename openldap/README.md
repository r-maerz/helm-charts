# OpenLDAP

This chart deploys an instance of OpenLDAP, based on the bitnami Docker image. It deliberatly excludes replication and high-availability config. If you require any of these features, kindly look at other charts such as [helm-openldap](https://github.com/jp-gouin/helm-openldap).

<br />

## Features

- LDAPS: Can set up and / or use a self-signed Certificate Authority
- LDAPS: Can be used with existing Let's Encrypt certificates (see below for required setup)
- Includes support for LDAP modules memberOf and refint
- Includes stakater/reloader as an optional dependency chart
- Does not include: Replication, High-Availability, LTB Password Reset, PHP LDAP Admin

<br />

## Setup

All setup options require at the bare minimum that a namespace is created and for you to add your basic LDAP structure as an LDIF in a kubernetes secret. The below assumes:

- namespace is called `iam`
- LDAP Domain / ORG is set to `another.com` / `dc=another,dc=com`
- LDAP Admin is called `admin`
- Two OUs (`groups` and `people`) and a single user (`Test User`) are added

<br />

### Create YAML files for namespace, environment variables and LDIF import
<br />

```bash
cat << EOF > ./openldap.namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: iam
  labels:
    name: iam
EOF
```

```bash
cat << 'EOF' > ./openldap.secret.env.yaml
apiVersion: v1
kind: Secret
metadata:
  name: openldap-secret-env
  namespace: iam
type: Opaque
stringData:
  LDAP_ADMIN_USERNAME: "admin"
  LDAP_ADMIN_PASSWORD: "ldap"
  LDAP_CONFIG_ADMIN_PASSWORD: "ldapconfig"
  LDAP_CONFIG_ADMIN_ENABLED: "yes"
  LDAP_ADMIN_DN: 'cn=admin,dc=another,dc=com'
  LDAP_ROOT: 'dc=another,dc=com'
EOF
```

```bash
cat << 'EOF' > ./openldap.secret.ldif-import.yaml
apiVersion: v1
kind: Secret
metadata:
  name: openldap-ldif-import
  namespace: iam
type: Opaque
stringData:
  00-root.ldif: |-
    dn: dc=another,dc=com
    objectClass: top
    objectClass: organization
    objectClass: domainRelatedObject
    objectClass: dcObject
    o: another COM ORG
    associatedDomain: another.com
  01-default-ous.ldif: |-
    dn: ou=people,dc=another,dc=com
    objectClass: organizationalUnit
    objectClass: extensibleObject
    ou: people

    dn: ou=groups,dc=another,dc=com
    objectClass: organizationalUnit
    ou: groups
  02-default-user.ldif: |-
    dn: cn=Test User,ou=people,dc=another,dc=com
    objectclass: inetOrgPerson
    objectclass: posixAccount
    objectclass: top
    cn: Test User
    displayName: Test User
    givenname: Test
    sn: User
    uid: test
    uidnumber: 1000
    gidnumber: 1000
    homedirectory: /home/users/test
    loginShell: /bin/bash
    mail: test@another.com
    userPassword: {SSHA}X0bmwsI3b4j/w1V44tE2pqpHc0bgu9xT
  03-default-groups.ldif: |-
    dn: cn=Admins,ou=groups,dc=another,dc=com
    description: All admin users
    cn: Admins
    objectClass: groupOfUniqueNames
    uniqueMember: cn=Test User,ou=people,dc=another,dc=com
EOF
```
<br />

### Apply all three manifests in order
<br />

`kubectl apply -f ./openldap.namespace.yaml`  
`kubectl apply -f ./openldap.secret.env.yaml`  
`kubectl apply -f ./openldap.secret.ldif-import.yaml`  
<br />

### OPTION A: Using a self-signed CA
<br />

The chart can create and use a self-signed CA. If you want to make use of that, create a `values.yaml`, adjusting the value of your LDAP Domain (`ldapDomain`) and Kubernetes secret which will hold the generated self-signed certificate (`ssl.secret.secretName`).  

> [!WARNING]  
> Ensure that the `ldapDomain` you provide in `values.yaml` matches the LDAP ORG you have configured in `openldap.secret.ldif-import.yaml`

<br />

```bash
cat << 'EOF' > ./openldap.values.yaml
ldapDomain: "another.com"

ssl:
  secret:
    secretName: another.com-tls
EOF
```
<br />
Now deploy the chart, pointing to the values file you just created.  

`helm upgrade --install openldap . --namespace iam --values ./openldap.values.yaml`  
<br />

### OPTION B: Use an existing TLS certificate, such as Let's Encrypt
<br />

When deploying in this mode, ensure you disable the selfsignedCA in your `values.yaml` and enable the letsencryptCA instead. Again, create your values file and make changes as necessary based on your LDAP ORG. Ensure the following is set as well:

<br />

```bash
cat << 'EOF' > ./openldap.values.yaml
ldapDomain: "another.com"

ssl:
  enabled: true
  secret:
    secretName: another.com-tls
  selfsignedCA: 
    enabled: false
    create: false
  letsencryptCA:
    enabled: true
EOF
```

> [!WARNING]  
> In this mode, `ssl.secret.secretName` has to point to a secret you have pre-created in the target namespace, containing at the bare minimum a `tls.crt` and `tls.key` entry. Let's Encrypt, when used with Cert-Manager, by default provides these.

<br />

Before you deploy the helm chart, figure out which of the Let's Encrypt Root CAs has issued your existing TLS cert [(see official Chain of Trust docs)](https://letsencrypt.org/certificates/). Once you know whether it is ISRG Root X1 or X2, create a Kubernetes secret in the target namespace with a .pem export of that Root CA.  

`kubectl create secret generic le-root-ca --from-literal "ca.crt=$(echo `curl -sL https://letsencrypt.org/certs/isrgrootx1.pem | tr -d '\n'`)" -n iam -o yaml > ./openldap.secret.le-prod-root-ca.yaml`  

<br />

Now proceed to deploy the chart.  

`helm upgrade --install openldap . --namespace iam --values ./openldap.values.yaml`