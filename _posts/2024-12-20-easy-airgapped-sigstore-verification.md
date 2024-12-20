# Introducing TrustRoot Assembler: Simplifying Airgapped Sigstore Verification

## Cluster Image Policies & custom Sigstore instances

Sigstore allows us to sign container images with the so called "keyless" method. The sigstore policy controller allows us to perform admission control of images in a Kubernetes cluster based on the validity of these keyless signatures. To be valid an image signature must be:
* integer: the image was not tampered and the signature matches it
* authentic: the signature was produced by a certain identity (e.g. a certain Github Workflow from a certain repository)

The `ClusterImagePolicy` Custom Resource allows us to define an admission policy that evaluates signature integrity (automatically) and authenticity (by checking that the oidc issuer in the signature certificate matches the issuer in `spec.keyless[x].identities[y].issuer`). Furthermore it allows us to define which signer identities are actually authorised (since anyone can sign with Sigstore!) by checking that the signature certificate identity matches that of `spec.keyless[x].identities[y].subject` or `spec.keyless[x].identities[y].subjectRegExp`. By default the controller assumes that a signature was produced using the so called Public Good instance of Sigstore (the one we interact with when using `cosign sign ...`)

```yaml
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: image-is-signed-by-github-actions-public-sigstore
spec:
  images:
  - glob: "**"
  authorities:
  - keyless:
      # Signed by the public Fulcio certificate authority
      url: https://fulcio.sigstore.dev
      identities:
      - issuer: https://token.actions.githubusercontent.com
        subjectRegExp: "https://github.com/falcorocks/.*/.github/workflows/.*@refs/heads/main"
    ctlog:
      # Signature stored into the public Rekor transparency log
      url: https://rekor.sigstore.dev
```

A field `spec.keyless[x].trustRootRef` can be used to inform the policy controller that the signature was produced using another Sigstore instance.

```yaml
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: image-is-signed-by-github-actions-my-sigstore
spec:
  images:
  - glob: "**"
  authorities:
  - keyless:
      # Signed by my private Fulcio certificate authority
      url: https://fulcio.my-sigstore.dev
      # Sigstore relies on my private Trust Root
      trustRootRef: my-sigstore
      identities:
      - issuer: https://token.actions.githubusercontent.com
        subjectRegExp:  "https://github.com/falcorocks/.*/.github/workflows/.*@refs/heads/main"
    ctlog:
      # Signature stored into my private Rekor transparency log
      url: https://rekor.my-sigstore.dev
```

When the default sigstore is used, the policy controller relies on the TrustRoot embedded in its source code. But if the `ClusterImagePolicy` specifies a `trustRootRef: my-sigstore` the policy controller will have to check for a `TrustRoot` Custom Resource that is called `my-sigstore` in the cluster. But what exactly is a `TrustRoot` anyways and why does it matter?

## TUF TrustRoots

Sigstore is a project that follows a zero trust philosphy. How does `cosign` know that it is talking to the right instance of `Fulcio`? How do we know that we are looking for a signature inside the right `Rekor` instance? In order for these components to trust each other, we need a PKI Root CA of sorts. The Update Framework (TUF) allows us to establish a central root of trust for these components to rely upon when making trust decisions. A TUF TrustRoot contains, among other things, the identities of the Sigstore components (fulcio, rekor, ct_log), a timestamp and the public keys of the TrustRoot "managers". This information is then signed by the private keys of the managers in a signing ceremony. There is a lot more to TUF and TrustRoot, so you should check [todo add link]. In practice, the easieast way to use TUF is to setup a TUF repository with TUF-ci, which is what Sigstore maintainers do. Cosign is shipped with an embedded TrustRoot, so it does not need to reach the TUF repository to get it. But eventually the repository is rotated, and then then the sigstore components have to regularly check for updates by querying the online repository.

The same applies to the policy controller. TrustRoot have to be updated and the process has to be online. The process can be fully automated in clusters that allow internet connectivity to the TrustRoot remote. But what about airgapped clusters? In that case the process has to be semi-automated: first we need to assemble an "offline" TrustRoot custom resource by reaching the mirror, then we deploy that custom resource in the airgapped cluster, where verification can happen fully offline.

The Sigstore documentation explains how to setup a `repository` TrustRoot but there are no clearly reproduceable examples. Inspired by prezha/TrustRoot I've created falcorocks/trustroot-assembler to easily assemble `repository` trustroots for any TUF instance. 