---
Title: OpenSSF and the sofware supply chain security
---

## The SSC security landscape

MITRE and OpenSSF increased their efforts to improve SSC security recently in general and in the open source.
MITRE for example proposes an end-to-end framework to preserve the integrity of the software supply chain. OpenSSF develops the SLSA framework that groups several seuciry best-practices for open source projects. The SLSA specification describes how to verify Provenance predicate to record the build characteristics of the produced artifacts as well as lays out other requirements that are to be verified separately. In detail, provenance enables to trace software back to the source and define the moving parts in a complex supply chain,

Moreover, the in-toto Attestation Framework defines a fixed, lightweight Statement that communicates some information about the execution of the sotware supply chain, such as if the source was reviewed, or if it went through a SLSA conformant build process.

This is where other in-toto predicate types can complement SLSA. in-toto allows for defining use case specific predicates that contain contextual information. Apart from SLSA Provenance, the framework currently lists definitions for in-toto Links, results of runtime traces, Software Bill of Materials (SBOM) specifications such as SPDX and CycloneDX, the Software Supply Chain Attribute Integrity (SCAI) specification, and SLSA Verification Summary Attestations.

Besides provenance attestations, in-toto allows to describe attestations for how to execute code review, run tests, runtime trace, and so on for a particular software.

## SSC safeguards

Safeguards against supply chain attacks can be classified by control type: directive, preventive, detective, corrective and recovery. But there ain't no such thing as a free lunch. Besides safeguards classification, the utility to cost ratio can also be an important factor to decide where to start from improving the supply chain seuciry of a software, and can be pretty easy to measure it.

There are cheap preventive safeguards that can be implemented in open source projects especially, where stakeholders platea can be pretty wide considering the contributions. For example branch protection rules are usually simple per-code repository configurations in providers (e.g. GitHub) and alongside pull-request based flows enforcing code review quorum, are also standard best practices nowadays. The same applies to reproducible builds, dependency pinning, build steps isolation, MFA authentication to repository providers, etc.

On the other side, safeguards with an high utility value can require a considerable amount of effort to be implemented from end-to-end (producer and consumer sides) - even though thanks to the already cited frameworks and related utilities are made easier.
Supply chain security is important to be constantly increased through safeguards placed as soon as possible in the supply chain ("shift left") and needs to be maintained in a continous manner reducing the operational burden ("DevSecOps") as technologies evolve and so attacks do.
 
Software artifacts signatures are one of the safeguards with a reasonable amount of effort nowadays, thanks also to important projects like Sigstore's Cosign for artifacts distributed as container images/artifacts. So this was one of the goals decided for the Falco Software Supply Chain Working Group.
