---
date: 2022-07-19T21:04:15+02:00
modified: 2022-07-19T21:06:50+02:00
title: GitOps multi tenancy notes
---

### Escalation temptative: impersonate privileged SA

Then, what if a tenant tries to escalate by using one of the Flux controllers privileged ServiceAccounts?

As spec.ServiceAccountName for Reconciliation resource cannot cross-namespace reference Service Accounts, tenants are able to let Flux apply his own resources only with ServiceAccounts that reside in his own Namespaces. Which is, Namespace of the ServiceAccount and Namespace of the Reconciliation resource must match.


### Escalation temptative: create and impersonate privileged SA

Then, what if a tenant tries to escalate by creating a privileged ServiceAccount inside on of his own Namespaces?

A tenant could create a ServiceAccount in an owned Namespace, but he can't neither bind at cluster-level ClusterRole nor in a non-owned Namespace as that wouldn't be permitted by Capsule admission controllers.

Now let's go on with the practical part.
