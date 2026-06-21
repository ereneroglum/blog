---
title: 'Migrating Keycloak Instances'
date: 2025-02-10T09:36:49+03:00
tags: [ 'keycloak', 'oidc', 'migration', 'realm', 'policy', 'role', 'oauth2', 'identity provider', 'idp' ]
---

Complete the following steps for each realm. There are 2 ways to do each step.
One via cli and one via web ui. If you are going to use the cli options, ensure
that at that step keycloak is not running.

First, export the realm via cli or web gui. For the cli option use:

```
kc export --realm (realm name) --file export.json
```

If you want to export all realms use:

```
kc export --file export.json
```

Only cli can be used to import if you have exported all realms.
Now transfer the exported file(s) and go to your second keycloak instance. You
can try to import the realm using cli or web ui. It is easier to spot errors
using cli import.

For cli import use:

```
kc import --file export.json --verbose
```

You will likely get errors while importing which will block the import proccess.
You can try to edit the json file to get rid of errors.

Common fixes for following errors include:

```
ERROR: Script upload is disabled
```

Remove or rewrite policies with ```"type": "js"```. If it is likely that these
policies are not used (such as default policies) you can remove them. If you are
removing a policy, search with its policy id to also remove roles depending on
the policy.

```
java.lang.RuntimeException: Error while importing policy [(policy name)]
```

It is likely that the policy depends on a role or a policy that does not exists.
Most likely you forgot to delete this policy when you were getting rid of
previous error. Either search with the name of the role and delete it or re-add
the role or rewrite the policy (keep in mind that you should not use
`"type": "js"`) when writing the policy.

```
org.keycloak.authorization.policy.provider.util.PolicyValidationException: Role can't be specified multiple times - (role name)
```

It is likely that your export file includes a policy with duplicate role. Search
with the role name and remove the dumplication in:

```
{
  "id": "(policy id)",
  "name": "(policy name)",
  "description": "(policy description)",
  "type": "role",
  "logic": "POSITIVE",
  "decisionStrategy": "UNANIMOUS",
  "config": {
    "roles": "[
      {\"id\":\"(role name)\", \"required\" : false },
      {\"id\":\"(role name)\", \"required\" : false }
    ]"
  }
},
```
