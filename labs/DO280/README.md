# Troubleshooting commands

- When a node has a taint it would typically have the effect of a pod remaining pending.
With oc events you would see if that is because of a taint, and which nodes have a taint. I find that the easiest way to remove a taint is the web console.

- man req EXAMPLES

- oc get nodes
- oc adm top node -l node-role.kubernetes.io/worker
- oc describe node ip-10-0-20-1-ap-southeast-1.computer.internel
- oc get pod -n openshift-image-registry
- oc logs --tail 3 -n openshift-iamge-registry cluster-image-registry-operator-xxxx-xxxx
- oc logs --tail 3 -n openshift-iamge-rgeistry -c cluster-image-registry-operator cluster-image-rgeistry-operator-xxxx-xxxx
- oc logs --tail 3 -n openshift-image-registry iamge-registry-xxx-yyyy
- oc adm node-logs --tail 3 -u kebelet ip-1.2.2..computer.internel
- oc debug node/ip-1.2.2..computer.internel
- oc status
- oc get events

# 3. Configuring Authentication

## commands


- htpasswd -C -B -b ~/temp admin USER_PASSWD
- htpasswd -b ~/temp developer USER_PASSWD
- cat ~temp
- oc login -u kuberadmin -p KUBEADM_PASSWD MASTER_API
- oc create secret generic localusers --from-file htpasswd=~/temp -n openshift-config
- oc adm policy add-cluster-role-to-user cluster-admin admin
- oc get -o yaml oauth cluster > ~/oauth.yaml

```
apiVersion: config.openshift.io/v1
kind: OAuth
...output omitted...
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: localusers
    mappingMethod: claim
    name: myusers
    type: HTPasswd
```
- oc replace -f ~/oauth.yaml

- oc login -u admin -p USER_PASSWD 
- oc get nodes
- oc login -u developer -p USER_PASSWD
- oc get nodes

```Error from server (Forbidden): nodes is forbidden: User "developer" cannot list
resource "nodes" in API group "" at the cluster scope
```
- oc login -u admin -p USER_P
- oc get users
- oc get identity

```
NAME                IDP NAME   IDP USER NAME   USER NAME
myusers:admin       myusers    admin           admin       ...
myusers:developer   myusers    developer       developer   ...
```
- oc extract secret/localusers -n openshift-config --to -
- htpasswd -b ~/temp manager USER_PASSWD
- cat ~/temp

```
admin:$2y$05$QPuzHdl06IDkJssT.tdkZuSmgjUHV1XeYU4FjxhQrFqKL7hs2ZUl6
developer:$apr1$0Nzmc1rh$yGtne1k.JX6L5s5wNa2ye.
manager:$apr1$CJ/tpa6a$sLhjPkIIAy755ZArTT5EH/
```

You must *update the secret* after *adding additional users*. Use the oc set data secret command to update the secret. If you receive a failure, then rerun the command again after a few moments as the oauth operator might still be reloading.

[student@workstation ~]$ oc set data secret/localusers \
>    --from-file htpasswd=/home/student/DO280/labs/auth-provider/htpasswd \
>    -n openshift-config
secret/localusers data updated
```

Create a new project named auth-provider, and then verify that the developer user cannot access the project

- oc login -u manager -p USER_PASSWD
- oc new-project auth-provider
- oc login -u developer -p USER_PASSWD
- oc delete project auth-provider

Error from server (Forbidden): projects.project.openshift.io "auth-provider"
is forbidden: User "developer" cannot delete resource "projects"
in API group "project.openshift.io" in the namespace "auth-provider"
```
- oc lgoin -u admin -p USER_PASSWD
- oc extract secret/localusers -n openshift-config --to -

Generate a random user password and assign it to the MANAGER_PASSWD variable.
[student@workstation ~]$ MANAGER_PASSWD="$(openssl rand -hex 15)"
Update the manager user to use the password stored in the MANAGER_PASSWD variable.
[student@workstation ~]$ htpasswd -b ~/DO280/labs/auth-provider/htpasswd \
>    manager ${MANAGER_PASSWD}

Updating password for user manager
htpasswd -b ~/DO280/labs/auth-provider/htpasswd \
>    manager ${MANAGER_PASSWD}
Update the secret.
[student@workstation ~]$ oc set data secret/localusers \
>    --from-file htpasswd=/home/student/DO280/labs/auth-provider/htpasswd \
>    -n openshift-config
secret/localusers data updated

Remove the manager user.
oc login -u admin -p redhat
oc extract secret/localusers -n openshift-config \
>    --to ~/DO280/labs/auth-provider/ --confirm
Delete manageer password
> htpasswd -D ~/DO280/labs/auth-provider/htpasswd manager
Update the secret.
[student@workstation ~]$ oc set data secret/localusers \
>    --from-file htpasswd=/home/student/DO280/labs/auth-provider/htpasswd \
>    -n openshift-config
secret/localusers data updated
Delete the identity resource for the manager user.
```
[student@workstation ~]$ oc delete identity "myusers:manager"
identity.user.openshift.io "myusers:manager" deleted
```
Delete the user resource for the manager user.

[student@workstation ~]$ oc delete user manager
user.user.openshift.io manager deleted

oc get users
NAME       UID                                   FULL NAME  IDENTITIES
admin      31f6ccd2-6c58-47ee-978d-5e5e3c30d617             myusers:admin
developer  d4e77b0d-9740-4f05-9af5-ecfc08a85101             myusers:developer

Remove the identity provider and clean up all users.
oc login -u kubeadmin -p ${RHT_OCP4_KUBEADM_PASSWD}
- oc edit oauth
```
Delete all the lines under spec:, and then append {} after spec:. Leave all the other information in the file unchanged. Your spec: line should match the following:

...output omitted...
spec: {}

- oc delete secret localusers -n openshift-config
- oc delete user --all
- oc delete identity --all
```

# Controlling Access to Openshift Resources


## Authorization Process
The authorization process is managed by rules, roles, and bindings.

|RBAC Object	| Description |
|-------------|:-----------|
|Rule|	Allowed actions for objects or groups of objects|
|Role	|Sets of rules. Users and groups can be associated with multiple roles.|
|Binding |	Assignment of users or groups to a role.|


RBAC Scope
Red Hat OpenShift Container Platform (RHOCP) defines two groups of roles and bindings depending on the scope and responsibility of users: cluster roles and local roles.

|Role Level	| Description |
|-----------|-------------|
|Cluster Role	| Users or groups with this role level can manage the OpenShift cluster.|
|Local Role	| Users or groups with this role level can only manage elements at a project level.|

## Managing RBAC Using the CLI

- oc adm policy add-cluster-role-to-user __cluster-role__ __username__
- oc adm policy remove-cluster-role-from-user __cluster-role__ __username__
- oc adm policy who-can delete user # delete user rule

## Default Roles
OpenShift ships with a set of default cluster roles that can be assigned locally or to the entire cluster. You can modify these roles for fine-grained access control to OpenShift resources, but additional steps are required that are outside the scope of this course.

| Default roles	|Description |
|---------------|------------|
|admin	| Users with this role can manage all project resources, including granting access to other users to the project.|
|basic-user |	Users with this role have read access to the project.|
|cluster-admin	| Users with this role have superuser access to the cluster resources. These users can perform any action on the cluster, and have full control of all projects.|
|cluster-status	| Users with this role can get cluster status information.|
| edit	| Users with this role can create, change, and delete common application resources from the project, such as services and deployment configurations. These users cannot act on management resources such as limit ranges and quotas, and cannot manage access permissions to the project.|
|self-provisioner	| Users with this role can create new projects. This is a cluster role, not a project role.|

- oc adm policy add-role-to-user role-name __username__ -n __project__
- oc adm policy add-role-to-user basic-user dev -n wordpress
view	Users with this role can view project resources, but cannot modify project resources.

## Users

### Regular users

This is the way most interactive OpenShift Container Platform users are represented. Regular users are created automatically in the system upon first login or can be created via the API. Regular users are represented with the User object. Examples: joe alice

### System users

Many of these are created automatically when the infrastructure is defined, mainly for the purpose of enabling the infrastructure to interact with the API securely. They include a cluster administrator (with access to everything), a per-node user, users for use by routers and registries, and various others. Finally, there is an anonymous system user that is used by default for unauthenticated requests. Examples: system:admin system:openshift-registry system:node:node1.example.com

### Service accounts

These are special system users associated with projects; some are created automatically when the project is first created, while project administrators can create more for the purpose of defining access to the contents of each project. Service accounts are represented with the ServiceAccount object. Examples: system:serviceaccount:default:deployer system:serviceaccount:foo:builder

## Groups

### system:authenticated

Automatically associated with all authenticated users.

### system:authenticated:oauth

Automatically associated with all users authenticated with an OAuth access token.

### system:unauthenticated

Automatically associated with all unauthenticated users.


## Exercice: Remove project creation privileges from users who are not OpenShift cluster administrators.

- oc get clusterrolebinding -o wide | grep -E 'NAME|self-provisioners'
- oc describe clusterrolebindings self-provisioners

```
Name:         self-provisioners
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  self-provisioner
Subjects:
  Kind   Name                        Namespace
  ----   ----                        ---------
  Group  system:authenticated:oauth
  ```

- oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
- oc describe clusterrolebindings self-provisioners
```
Error from server (NotFound): clusterrolebindings.rbac.authorization.k8s.io "self-provisioners" not found
```

Log in as the leader user with a password of redhat, and then try to create a project. Project creation should fail.

- oc login -u leader -p redhat
- oc new-project test
  
  Error from server (Forbidden): You may not request a new project via this API.

Create a project and add project administration privileges to the leader user.

- oc login -u admin -p redhat
- oc new-project auth-rbac

Grant project administration privileges to the leader user on the auth-rbac project.
- oc policy add-role-to-user admin leader

Create the dev-group and qa-group groups and add their respective members
- oc adm groups new dev-group
- oc adm groups add-users dev-group developer
- oc adm groups new qa-group
- oc adm groups add-users qa-group qa-engineer
- oc get groups

As the leader user, assign write privileges for dev-group and read privileges for qa-group to the auth-rbac project.
- oc policy add-role-to-group edit dev-group
- oc policy add-role-to-group view qa-group
- oc get rolebindings -o wide

```
NAME      ROLE                AGE   USERS    GROUPS      ...
admin     ClusterRole/admin   58s   admin
admin-0   ClusterRole/admin   51s   leader
edit      ClusterRole/edit    12s            dev-group
...output omitted...
view      ClusterRole/view    8s             qa-group
```

As the developer user, deploy an Apache HTTP Server to prove that the developer user has write privileges in the project. Also try to grant write privileges to the qa-engineer user to prove that the developer user has no project administration privileges.
- oc login -u developer -p developer
- oc new-app --name httpd httpd:2.4
--> Success
- oc policy add-role-to-user edit qa-engineer
Error from server (Forbidden): rolebindings.rbac.authorization.k8s.io is forbidden: User "developer" cannot list resource "rolebindings" in API group "rbac.authorization.k8s.io" in the namespace "auth-rbac"

Restore project creation privileges to all users.
- oc login -u admin -p redhat

Restore project creation privileges for all users by recreating the self-provisioners cluster role binding created by the OpenShift installer. You can safely ignore the warning that the group was not found.
- oc adm policy add-cluster-role-to-group --rolebinding-name self-provisioners self-provisioner system:authenticated:oauth


## Lab: Verifying the Health of a Cluster
In this lab, you will configure the HTPasswd identity provider, create groups, and assign roles to users and groups.

Outcomes
- Create users and passwords for HTPasswd authentication.
- Configure the Identity Provider for HTPasswd authentication.
- Assign cluster administration rights to users.
- Remove the ability to create projects at the cluster level.
- Create groups and add users to groups.
- Manage user privileges in projects by granting privileges to groups.

---

- Update the existing ~/DO280/labs/auth-review/tmp_users HTPasswd authentication file to remove the analyst user. Ensure that the tester and leader users in the file use a password of L@bR3v!ew. Add two new entries to the file for the users admin and developer. Use L@bR3v!ew as the password for each new user.

```
htpasswd -D ~/DO280/labs/auth-review/tmp_users analyst

for NAME in tester leader admin developer
>    do
>    htpasswd -b ~/DO280/labs/auth-review/tmp_users ${NAME} 'L@bR3v!ew'
>    done

cat ~/DO280/labs/auth-review/tmp_users
tester:$apr1$0eqhKgbU$DWd0CB4IumhasaRuEr6hp0
leader:$apr1$.EB5IXlu$FDV.Av16njlOCMzgolScr/
admin:$apr1$ItcCncDS$xFQCUjQGTsXAup00KQfmw0
developer:$apr1$D8F1Hren$izDhAWq5DRjUHPv0i7FHn.
```

- Log in to your OpenShift cluster as the kubeadmin user using the RHT_OCP4_KUBEADM_PASSWD variable defined in the /usr/local/etc/ocp4.config file as the password. Configure your cluster to use the HTPasswd identity provider using the user names and passwords defined in the ~/DO280/labs/auth-review/tmp_users file.

```
oc login -u kubeadmin -p ${RHT_OCP4_KUBEADM_PASSWD} \
>    https://api.ocp4.example.com:6443

oc create secret generic auth-review \
>    --from-file htpasswd=/home/student/DO280/labs/auth-review/tmp_users \
>    -n openshift-config

oc get oauth cluster \
>    -o yaml > ~/DO280/labs/auth-review/oauth.yaml

- Edit the ~/DO280/labs/auth-review/oauth.yaml file to replace the spec: {} line with the following bold lines. Note that htpasswd, mappingMethod, name and type are at the same indentation level.

apiVersion: config.openshift.io/v1
kind: OAuth
...output omitted...
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: auth-review
    mappingMethod: claim
    name: htpasswd
    type: HTPasswd

oc replace -f ~/DO280/labs/auth-review/oauth.yaml

watch oc get pods -n openshift-authentication
Every 2.0s: oc get pods -n openshift-authentication            ...

NAME                              READY   STATUS    RESTARTS   AGE
oauth-openshift-6755d8795-h8bgv   1/1     Running   0          34s
oauth-openshift-6755d8795-rk4m6   1/1     Running   0          38s
```

- Make the admin user a cluster administrator. Log in as both admin and as developer to verify HTPasswd user configuration and cluster privileges.

```
oc adm policy add-cluster-role-to-user \
>    cluster-admin admin

oc login -u admin -p 'L@bR3v!ew'

oc get nodes
NAME       STATUS   ROLES           AGE   VERSION
master01   Ready    master,worker   46d   v1.19.0+d856161
master02   Ready    master,worker   46d   v1.19.0+d856161
master03   Ready    master,worker   46d   v1.19.0+d856161

oc login -u developer -p 'L@bR3v!ew'

oc get nodes
Error from server (Forbidden): nodes is forbidden: User "developer" cannot list
resource "nodes" in API group "" at the cluster scope
```

- 

- As the admin user, remove the ability to create projects cluster wide.

```
oc login -u admin -p 'L@bR3v!ew'

Remove the self-provisioner cluster role from the system:authenticated:oauth virtual group.

oc adm policy remove-cluster-role-from-group  \
>    self-provisioner system:authenticated:oauth
```

- Create a group named managers, and add the leader user to the group. Grant **project creation** privileges to the managers group. As the leader user, create the auth-review project.

```
oc adm groups new managers

oc adm groups add-users managers leader

oc adm policy add-cluster-role-to-group  \
>    self-provisioner managers

oc login -u leader -p 'L@bR3v!ew'
oc new-project auth-review
```

- Create a group named developers and grant edit privileges on the auth-review project. Add the developer user to the group.

```
oc login -u admin -p 'L@bR3v!ew'

oc adm groups new developers
oc adm groups add-users developers developer

oc policy add-role-to-group edit developers

```

- Create a group named qa and grant view privileges on the auth-review project. Add the 
tester user to the group.

```
oc adm groups new qa
oc adm groups add-users qa tester
oc policy add-role-to-group view qa
```

## Managing Sensitive Information with Secrets

### Creating a Secret

- Create a generic secret containing key-value pairs
```
oc create secret generic secret_name \
>    --from-literal key1=secret1 \
>    --from-literal key2=secret2
```

- Create a generic secret
```
oc create secret generic ssh-keys \
>    --from-file id_rsa=/path-to/id_rsa \
>    --from-file id_rsa.pub=/path-to/id_rsa.pub
```

- Create a TLS secret
```
oc create secret tls secret-tls \
>    --cert /path-to-certificate --key /path-to-key
```

- oc create secret generic __secret_name__ --from-literal key1=secret1 --from-literal key2=secret2
- Update the pod service account to allow the reference to the secret. to allow a secret to be mounted by a pod running under a specific service account
- oc secrets add --for mount serviceaccount/__serviceaccount-name__ secret/__secret_name__

### Secrets as Pod Environment Variables
```
env:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: demo-secret
        key: root_password
```        

- oc set env dc/demo --from=secret/__demo-secret__

### Secrets as Files in a Pod
- oc set volume dc/demo --add --type=secret --secret-name=demo-secret --mount-path=/app-secrets

- oc create secret generic mysql --from-literal user=myuser --from-literal password=redhat123 --from-literal database=test_secrets --from-literal hostname=mysql 
- oc new-app --name mysql --docker-image registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-47
- oc set env dc/mysql --prefix MYSQL_ --from secret/mysql


oc set env deployment/demo --from secret/demo-secret \
>    --prefix MYSQL_

oc set volume deployment/demo \ 
>    --add --type secret \ 
>    --secret-name demo-secret \ 
>    --mount-path /app-secrets 
```
spec:
      containers:
      - env:
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              key: database
              name: mysql
        - name: MYSQL_HOSTNAME
          valueFrom:
            secretKeyRef:
              key: hostname
              name: mysql
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              key: password
              name: mysql
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              key: user
              name: mysql
```
- oc rsh mysql-2-rqp77
```
sh-4.2$ mysql -u myuser --password=redhat123 test_secrets -e 'show databases;'
sh-4.2$ exit
```
- oc new-app --name quotes --docker-image quay.io/redhattraining/famous-quotes:1.0

- oc get pods -l app=quotes
- oc set env dc/quotes --prefix QUOTES_ --from secret/mysql


## Controlling Application Permissions with Security Context Constraints(SCCs)

Red Hat OpenShift provides security context constraints (SCCs), a security mechanism that restricts access to resources, but not to operations in OpenShift.

SCCs limit the access from a running pod in OpenShift to the host environment. SCCs control:

- Running privileged containers
- Requesting extra capabilities to a container
- Using host directories as volumes
- Changing the SELinux context of a container
- Changing the user ID

```
$ oc get scc
OpenShift provides eight SCCs:

anyuid

hostaccess

hostmount-anyuid

hostnetwork

node-exporter

nonroot

privileged

restricted
$  oc describe scc anyuid
Name:           anyuid
Priority:         10
Access:
  Users:          <none>
  Groups:         system:cluster-admins
Settings:
  Allow Privileged:       false
  Default Add Capabilities:     <none>
  Required Drop Capabilities:     MKNOD,SYS_CHROOT
  Allowed Capabilities:       <none>
  Allowed Volume Types:       configMap,downwardAPI,emptyDir,persistentVolumeClaim,secret
  Allow Host Network:       false
  Allow Host Ports:       false
  Allow Host PID:       false
  Allow Host IPC:       false
  Read Only Root Filesystem:      false
  Run As User Strategy: RunAsAny
    UID:          <none>
    UID Range Min:        <none>
    UID Range Max:        <none>
  SELinux Context Strategy: MustRunAs
    User:         <none>
    Role:         <none>
    Type:         <none>
    Level:          <none>
  FSGroup Strategy: RunAsAny
    Ranges:         <none>
  Supplemental Groups Strategy: RunAsAny
    Ranges:         <none>
```
- oc create serviceaccount __service-account-name__
- oc adm policy add-scc-to-user scc -z __service-account-name__
- oc new-project authorization-scc
- oc new-app --name gitlab --docker-image gitlab/gitlab-ce:8.4-ce.0
- oc get pods
- oc logs pod/gitlab-1-lasdf
```
...output omitted...
================================================================================
Recipe Compile Error in /opt/gitlab/embedded/cookbooks/\
cache/\cookbooks/gitlab/recipes/default.rb
================================================================================

Chef::Exceptions::InsufficientPermissions
-----------------------------------------
directory[/etc/gitlab] (gitlab::default line 26) had an error: \
Chef::Exceptions::InsufficientPermissions: Cannot create directory[/etc/gitlab] at \
/etc/gitlab due to insufficient permissions
...output omitted...
```
- oc create sa gitlab-sa
- oc login -u admin -p XXXXXX
- oc adm policy add-scc-to-user anyuid -z gitlab-sa
- oc login -u developer -p XXX
- oc set serviceaccount deploymentconfig gitlab gitlab-sa
- oc get pods
- oc expose service gitlab --port 80
- oc get route gitlab

## view Roles and Users for a project
```
$ oc get rolebindings
NAME                    ROLE                    USERS     GROUPS                                 SERVICE ACCOUNTS   SUBJECTS
system:image-pullers    /system:image-puller              system:serviceaccounts:asdfasdf4asdf
admin                   /admin                  jsmith
system:deployers        /system:deployer                                                         deployer
system:image-builders   /system:image-builder   
```
## view Roles and Users for the Cluster
```
$ oc get clusterrolebindings
NAME                                            ROLE                                       USERS           GROUPS                                         SERVICE ACCOUNTS                                   SUBJECTS
system:job-controller                           /system:job-controller                                                                                    openshift-infra/job-controller
system:build-controller                         /system:build-controller                                                                                  openshift-infra/build-controller
system:node-admins                              /system:node-admin                         system:master   system:node-admins
registry-registry-role                          /system:registry                                                                                          default/registry
system:pv-provisioner-controller                /system:pv-provisioner-controller                                                                         openshift-infra/pv-provisioner-controller
basic-users                                     /basic-user                                                system:authenticated
system:namespace-controller                     /system:namespace-controller                                                                              openshift-infra/namespace-controller
system:discovery-binding                        /system:discovery                                          system:authenticated, system:unauthenticated
system:build-strategy-custom-binding            /system:build-strategy-custom                              system:authenticated
cluster-status-binding                          /cluster-status                                            system:authenticated, system:unauthenticated
system:webhooks                                 /system:webhook                                            system:authenticated, system:unauthenticated
system:gc-controller                            /system:gc-controller                                                                                     openshift-infra/gc-controller
cluster-readers                                 /cluster-reader                                            system:cluster-readers
system:pv-recycler-controller                   /system:pv-recycler-controller                                                                            openshift-infra/pv-recycler-controller
system:daemonset-controller                     /system:daemonset-controller                                                                              openshift-infra/daemonset-controller
cluster-admins                                  /cluster-admin                             system:admin    system:cluster-admins
system:hpa-controller                           /system:hpa-controller                                                                                    openshift-infra/hpa-controller
system:build-strategy-source-binding            /system:build-strategy-source                              system:authenticated
system:replication-controller                   /system:replication-controller                                                                            openshift-infra/replication-controller
system:sdn-readers                              /system:sdn-reader                                         system:nodes
system:build-strategy-docker-binding            /system:build-strategy-docker                              system:authenticated
system:routers                                  /system:router                                             system:routers
system:oauth-token-deleters                     /system:oauth-token-deleter                                system:authenticated, system:unauthenticated
system:node-proxiers                            /system:node-proxier                                       system:nodes
system:nodes                                    /system:node                                               system:nodes
self-provisioners                               /self-provisioner                                          system:authenticated:oauth
system:service-serving-cert-controller          /system:service-serving-cert-controller                                                                   openshift-infra/service-serving-cert-controller
system:registrys                                /system:registry                                           system:registries
system:pv-binder-controller                     /system:pv-binder-controller                                                                              openshift-infra/pv-binder-controller
system:build-strategy-jenkinspipeline-binding   /system:build-strategy-jenkinspipeline                     system:authenticated
system:deployment-controller                    /system:deployment-controller                                                                             openshift-infra/deployment-controller
system:masters                                  /system:master                                             system:masters
system:service-load-balancer-controller         /system:service-load-balancer-controller    
```


# Chapter 5. Configuring OpenShift Networking Components

## OpenShift Software-defined Networking
- oc describe dns.operator/default
- oc get  Network.config.openshift.io cluster 

## Controlling Cluster Network Ingress

### Describing Methods for Managing Ingress Traffic
- Ingress (resource). The Ingress Operator manages this resource. Ingresses accept external requests and proxy them based on the route. You can only route HTTP, HTTPS and server name identification (SNI), and TLS with SNI.

- External load balancer (service type). This resources instructs OpenShift to spin up a load balancer in a cloud environment. A load balancer instructs OpenShift to interact with the cloud provider in which the cluster is running to provision a load balancer.

- Service external IP (service type). This method instructs OpenShift to set NAT rules to redirect traffic from one of the cluster IPs to the container.

- NodePort (service type). With this method, OpenShift exposes a service on a static port on the node IP address. You must ensure that the external IP addresses are properly routed to the nodes.

### Describing Route Options and Route Types
#### OpenShift Secure Routes
- Edge
- Pass-through
- Re-encryption
-  oc create route edge --service api-frontend --api.apps.acme.com --key api.key --cert api.crt
#### Creating Insecure Routes
-  oc expose service api-frontend

### extra notes 
- If you have HTTP/HTTPS, use an Ingress Controller.
- If you have a TLS-encrypted protocol other than HTTPS. For example, for TLS with the SNI header, use an Ingress Controller.
- Otherwise, use a Load Balancer, an External IP, or a *NodePort*.

### create CSR and Key

#### generates an RSA private key

- openssl genrsa -out `training.key` 2048

#### generate certificate signing request CSR

- openssl req -new -key `training.key` -out training.csr

#### generate signed certificate

- openssl x509 -req -in `training.csr` -passin file:passphrase.txt -CA training-CA.pem -CAkey 
