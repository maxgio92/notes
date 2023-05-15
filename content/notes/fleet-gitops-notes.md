---
Title: Notes on Fleet
---

* Multi-cluster built-in
* Installations convert automatically no matter the resource types of which the manifests, everything to Helm releases
* Helm release declarations are not also Custom Resources, but only local files (`fleet.yaml`).
* At the time of https://www.youtube.com/watch?v=rIH_2CUXmwM Fleet didn't support upward paths (like ".."), essential for using it with Kustomize.
* Fleet seems to not detect drifts and thus it doesn't remove them when happening.
