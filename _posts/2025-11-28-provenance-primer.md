---
title: "Software Supply Chain Build Provenance: a primer"
date: 2025-11-28
---

If you work in software security, you've likely watched the rise of supply chain attacks with growing concern. From SolarWinds to Codecov, attackers have realized it's often easier to compromise the build pipeline than the application itself. When a malicious package sneaks into your dependency tree or a compromised artifact lands in production, our traditional security controls often stay silent until it's too late.

Provenance attestation is the answer to this problem. It provides cryptographically verifiable metadata about how, when, where, and from what your software was built. Think of it as a "birth certificate" for your artifacts. In this article, we'll look at what provenance attestation is, why the old tools aren't enough, and how it changes the software supply chain security game.

## Why Provenance?

We're all familiar with authenticity and integrity. When you verify a digital signature or check a file's hash, you're asking two questions: "Did this come from who I think it came from?" and "Has it been tampered with?" These are essential checks. They tell you to trust an artifact based on its current state. But what if the origin itself is the problem?

Imagine an attacker compromising your build server to inject malicious code. The resulting binary will be signed with your valid key (authenticity check: pass) and won't change after signing (integrity check: pass). But it's still malicious. Authenticity and integrity confirm the artifact's state, but they can't answer the deeper question: "What is the true origin of this artifact?"

This is the gap provenance attestation fills. While authenticity asks "who signed this?", provenance asks "where did this really come from?" It doesn't replace the old checks; it completes them. Together, they give you the full picture: you know who created it, that it hasn't changed, and exactly how it came to be.

## What is Provenance?

Let's strip this down. Building software is just a transformation. You put source code in, you get an artifact out. Whether you're running `go build` to create a binary, `docker build` to create a container image, or `npm publish` to package a library, the pattern is identical. This magic usually happens on a platform which usually means a CI system like GitHub Actions or Jenkins.

The industry standard for capturing this process is the in-toto provenance specification, part of the SLSA (Supply-chain Levels for Software Artifacts) framework. SLSA (pronounced "salsa") is a ladder of security guarantees. At Level 2 and above, it combines three critical things into one signed document:
1.  **Authenticity**: A cryptographic signature verifying *who* created the document.
2.  **Integrity**: A digest of the artifact, so you can verify it matches *what* was built.
3.  **Provenance**: The detailed story of *where*,*how* and *from what* it was built.

Here's what a real one looks like (simplified for sanity):

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [{
    "name": "myapp",
    "digest": {"sha256": "abc123..."}
  }],
  "predicateType": "https://slsa.dev/provenance/v1",
  "predicate": {
    "buildDefinition": {
      "buildType": "https://actions.github.io/buildtypes/workflow/v1",
      "externalParameters": {
        "workflow": {
          "repository": "https://github.com/example/myapp",
          "path": ".github/workflows/build.yml",
          "ref": "refs/heads/main"
        }
      },
      "internalParameters": {
        "github": {
          "event_name": "push",
          "repository_id": "123456",
          "runner_environment": "github-hosted"
        }
      },
      "resolvedDependencies": [{
        "uri": "git+https://github.com/example/myapp@refs/heads/main",
        "digest": {"sha1": "def456..."}
      }]
    },
    "runDetails": {
      "builder": {
        "id": "https://github.com/actions/workflows/build.yml@refs/heads/main"
      },
      "metadata": {
        "invocationId": "run-123",
        "startedOn": "2024-01-15T10:30:00Z"
      }
    }
  }
}
```

The git commit digest tells you exactly which version of the source code was used, eliminating ambiguity about what code went into the build. The branch and repository information show where the code came from, allowing you to enforce policies like "only accept builds from the protected main branch." The builder infrastructure information reveals whether the build ran on a GitHub-hosted runner or a self-hosted runner.

## Provenance Based Verification Model

Here is the hard truth: verifying integrity and authenticity is necessary, but it's not enough. Those checks only tell you that you have a valid, unaltered file signed by a trusted key. They answer "Is this legit?" but fail to answer "Should I trust it?" To answer that, we need to compare the provenance metadata against our **expectations**. Think of these expectations as an authorization barrier for your code.

Consider a malicious insider with access to your signing keys. They could build a malicious artifact from a personal feature branch and sign it. Cryptographically, it's perfect. But the provenance metadata tells the real story: the build came from `feature/malicious-branch` instead of `main`. A simple policy check would catch this immediately, rejecting the artifact despite its valid signature.

This verification model works well in these scenarios:

- **Deployment admission control**: Ensuring only approved code runs in production.
- **Transitive trust**: Verifying the security of base images and dependencies.
- **Privileged tooling**: Checking that the tools used in sensitive operations are trustworthy.

In deployment admission control you can enforce strict origin policies. For example, you might only allow artifacts built from specific repositories, or demand that they come from protected branches that require changes to go through PRs with mandatory code review. A Kubernetes admission controller can verify these details before allowing a pod to start. It could check that a container image was built from the main branch using a GitHub-hosted runner. Even if an attacker signs a malicious image, the provenance will reveal the unauthorized branch, and the admission controller will reject it.

When building on top of other artifacts, provenance enables transitive trust verification. Consider a Dockerfile that uses `FROM base-image:latest`. Before building your application, you can verify that the base image's provenance shows it came from the expected origin. This prevents supply chain attacks where a compromised base image propagates into your builds. The provenance attestation of the base image becomes part of your build's dependency chain, allowing you to trace the entire lineage of your final artifact back to trusted sources.

Another critical scenario is running tools with high privileges. Whenever you execute operations that are sensitive, such as authentication or system configuration, you should verify that the tools themselves come from expected origins. For example, before using the Cosign CLI to sign artifacts or verify attestations, you should check that the Cosign binary itself has provenance showing it was built from the official Sigstore repository. This is especially important in automations, for instance in CI workflows run by bots.

## Conclusion

Provenance attestation is more than just a security upgrade; it's a mindset shift. We're moving from a world where we blindly trust signatures to one where we verify origins. It's the difference between checking an ID card and verifying a background check. By combining authenticity, integrity, and provenance, we can build authorization barriers that reject even correctly signed artifacts if they don't come from trusted processes. As attackers get smarter, our trust models need to get stricter. The question isn't "who signed this?", but "how was this built?".

However, provenance alone isn't a silver bullet. If the build platform generating the provenance is compromised, it can be tricked into lying. In the next article, we'll explore attacks against provenance generation itself and how higher SLSA levels help mitigate them.
