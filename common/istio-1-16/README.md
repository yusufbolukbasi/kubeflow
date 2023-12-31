# Istio

## Upgrade Istio Manifests

Istio ships with an installer called `istioctl`, which is a deployment /
debugging / configuration management tool for Istio all in one package.
In this section, we explain how to upgrade our istio kustomize packages
by leveraging `istioctl`. Assuming the new version is `X.Y.Z` and the
old version is `X1.Y1.Z1`:

1.  Make a copy of the old istio manifests tree, which will become the
    kustomization for the new Istio version:

        $ export MANIFESTS_SRC=<path/to/manifests/repo>
        $ export ISTIO_OLD=$MANIFESTS_SRC/common/istio-X1-Y1
        $ export ISTIO_NEW=$MANIFESTS_SRC/common/istio-X-Y
        $ cp -a $ISTIO_OLD $ISTIO_NEW

2.  Download `istioctl` for version `X.Y.Z`:

        $ ISTIO_VERSION="X.Y.Z"
        $ wget "https://github.com/istio/istio/releases/download/${ISTIO_VERSION}/istio-${ISTIO_VERSION}-linux.tar.gz"
        $ tar xvfz istio-${ISTIO_VERSION}-linux.tar.gz
        # sudo mv istio-${ISTIO_VERSION}/bin/istioctl /usr/local/bin/istioctl

3.  Use `istioctl` to generate an `IstioOperator` resource, the
    CustomResource used to describe the Istio Control Plane:

        $ cd $ISTIO_NEW
        $ istioctl profile dump default > profile.yaml

    ---
    **NOTE**

    `istioctl` comes with a bunch of [predefined
    profiles](https://istio.io/v1.9/docs/setup/additional-setup/config-profiles/)
    (`default`, `demo`, `minimal`, etc.). The `default` profile is installed by default.

    ---

4.  Generate manifests and add them to their respective packages. We
    will generate manifests using `istioctl`, the
    `profile.yaml` file from upstream and the
    `profile-overlay.yaml` file that contains our desired
    changes:

        $ export PATH="$MANIFESTS_SRC/scripts:$PATH"
        $ cd $ISTIO_NEW
        $ istioctl manifest generate --cluster-specific -f profile.yaml -f profile-overlay.yaml > dump.yaml
        $ split-istio-packages -f dump.yaml
        $ mv $ISTIO_NEW/crd.yaml $ISTIO_NEW/istio-crds/base
        $ mv $ISTIO_NEW/install.yaml $ISTIO_NEW/istio-install/base
        $ mv $ISTIO_NEW/cluster-local-gateway.yaml $ISTIO_NEW/cluster-local-gateway/base

    ---
    **NOTE**

    `split-istio-packages` is a python script in the same folder as this file.
    The `ruamel.yaml` version used is 0.16.12.

    `--cluster-specific` is a flag that determines if a current K8s cluster context will be used to dynamically 
    detect default settings. Ensure you have a target cluster ready before running the above commands. 
    We set this flag because `istioctl manifest generate` generates manifest files with resources that are no 
    longer supported in Kubernetes 1.25 (`policy/v1beta1`). See: https://github.com/istio/istio/issues/41220

    ---

5. Remove PodDisruptionBudget from `istio-install` and `cluster-local-gateway` kustomizations. 
   See https://github.com/istio/istio/issues/12602 and https://github.com/istio/istio/issues/24000

   Until now we have used two patches:
   - `common/istio-1-16/istio-install/base/patches/remove-pdb.yaml`
   - `common/istio-1-16/cluster-local-gateway/base/patches/remove-pdb.yaml`
   
   The above patches do not work with kustomize v3.2.0 as it doesn't have the appropriate 
   openapi schemas for the policy/v1 API version resources. This is fixed in kustomize v4+. 
   See https://github.com/kubernetes-sigs/kustomize/issues/3694#issuecomment-799700607 and
   https://github.com/kubernetes-sigs/kustomize/issues/4495
   
   A temporary workaround is to use the following instructions to manually delete the PodDisruptionBudget resources with `yq`: 
        
        $ yq eval -i 'select((.kind == "PodDisruptionBudget" and .metadata.name == "cluster-local-gateway") | not)' common/istio-1-16/cluster-local-gateway/base/cluster-local-gateway.yaml
        $ yq eval -i 'select((.kind == "PodDisruptionBudget" and .metadata.name == "istio-ingressgateway") | not)' common/istio-1-16/istio-install/base/install.yaml
        $ yq eval -i 'select((.kind == "PodDisruptionBudget" and .metadata.name == "istiod") | not)' common/istio-1-16/istio-install/base/install.yaml
        
   ---
   **NOTE**
   
   NOTE: Make sure to remove a redundant {} at the end of the `common/istio-1-16/istio-install/base/install.yaml` and `common/istio-1-16/cluster-local-gateway/base/cluster-local-gateway.yaml` files.
   
   ---

## Changes to Istio's upstream manifests

### Changes to the upstream IstioOperator profile

Changes to Istio's upstream profile `default` are the following:

-   Add a `cluster-local-gateway` component for KFServing.
-   Disable the EgressGateway component. We don't use it and it adds
    unnecessary complexity.

Those changes are captured in the [profile-overlay.yaml](profile-overlay.yaml)
file.

### Changes to the upstream manifests using kustomize

The Istio kustomizations make the following changes:

- Remove PodDisruptionBudget from `istio-install` and `cluster-local-gateway` kustomizations. See:
    - https://github.com/istio/istio/issues/12602
    - https://github.com/istio/istio/issues/24000
- Add EnvoyFilter for adding an `X-Forwarded-For` header in requests passing through the Istio Ingressgateway, inside the `istio-install` kustomization.
- Add Istio AuthorizationPolicy to allow all requests to the Istio Ingressgateway and the Istio cluster-local gateway.
- Add Istio AuthorizationPolicy in Istio's root namespace, so that sidecars deny traffic by default (explicit deny-by-default authorization model).
- Add Gateway CRs for the Istio Ingressgateway and the Istio cluster-local gateway, as `istioctl` stopped generating them in later versions.
- Add the istio-system namespace object to `istio-namespace`, as `istioctl` stopped generating it in later versions.
- Configure TCP KeepAlives.
- Disable tracing as it causes DNS breakdown. See:
  https://github.com/istio/istio/issues/29898
