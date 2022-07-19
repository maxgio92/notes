---
date: 2022-07-19T21:04:15+02:00
modified: 2022-07-19T21:04:49+02:00
title: GitOps multi tenancy notes
---

Escalation temptative: create and impersonate privileged SA

Then, what if a tenant tries to escalate by creating a privileged ServiceAccount inside on of his own Namespaces?

A tenant could create a ServiceAccount in an owned Namespace, but he can't bind at cluster-level ClusterRole as that wouldn't be permitted by Capsule admission controllers.

Now let's go on with the practical part.
