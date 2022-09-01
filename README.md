# NVIDIA Morpheus SDK

This Helm chart deploys the NVIDIA Morpheus SDK container image.

## Overview

NVIDIA Morpheus is an open AI application framework that provides cybersecurity developers with a highly optimized AI pipeline and pre-trained AI capabilities that, for the first time, allow them to instantaneously inspect all IP traffic across their data center fabric. Bringing a new level of security to data centers, Morpheus provides dynamic protection, real-time telemetry, adaptive policies, and cyber defenses for detecting and remediating cybersecurity threats.

### Setup

The Morpheus SDK container is packaged as a [Kubernetes](https://kubernetes.io/docs/home/) (aka k8s) deployment using a [Helm](https://helm.sh/docs/) chart. NVIDIA provides installation instructions for the [NVIDIA Cloud Native Core Stack](https://github.com/NVIDIA/cloud-native-core) which incorporates the setup of these platforms and tools. Morpheus and its use of Triton Inference Server are initially designed to use the T4 (the G4 instance type in AWS EC2), V100 (P3), or A100 family of GPU (P4d).

#### NGC API KEY

First, you will need to set up your NGC API Key to access all the Morpheus components, using the instructions from the [NGC Registry CLI User Guide](https://docs.nvidia.com/dgx/ngc-registry-cli-user-guide/index.html#topic_3). Once you have created your API key, create an environment variable containing your API key for use by the commands used further in these instructions:

```bash
export API_KEY="<your key>"
```

After installing the Cloud Native Core Stack, install and configure the NGC Registry CLI using the instructions from the [NGC Registry CLI User Guide](https://docs.nvidia.com/dgx/ngc-registry-cli-user-guide/index.html#topic_3).

#### CREATE NAMESPACE FOR MORPHEUS

Create a namespace and an environment variable for the namespace to organize the k8s cluster deployed via EGX Stack and logically separate Morpheus-related deployments from other projects using the following command:

```bash
kubectl create namespace <some name>
export NAMESPACE="<some name>"
```

### Install the Morpheus SDK CLI

```bash
helm fetch https://helm.ngc.nvidia.com/nvidia/morpheus/charts/morpheus-sdk-client-22.06.tgz --username='$oauthtoken' --password=$API_KEY --untar
helm install --set ngc.apiKey="$API_KEY" \
 --set sdk.args="/bin/sleep infinity" \
 --namespace $NAMESPACE \
 sdk1 \
 morpheus-sdk-client
```

### Chart values explained

Various fields required to access the container images from NGC. For the public catalog images, it should be sufficient to just specify the provided username and your API_KEY.

```yaml
ngc:
  username: "$oauthtoken"
  apiKey: ""
  org: ""
  team: ""
```

The identity of the public catalog Morpheus SDK client image which could be overridden for other registry locations. As noted in the AI Engine chart overview, the withEngine flag specifies node affinity with the ai-engine deployment which will provide the most optimal performance since the SDK pod and the Triton pod can share the CUDA device pool memory of the GPU. The default args for the pod are to put it to sleep which means a user can use `kubectl exec -it <sdk pod name> -- bash` to run Morpheus pipelines interactively or just inspect the contents.

```yaml
sdk:
  registry: "nvcr.io/nvidia/morpheus"
  image: morpheus
  version: 22.06-runtime
  withEngine: true
  args: "/bin/sleep infinity"
  # args: "morpheus --help"
  # args: "morpheus run pipeline-nlp ..."
```

A local host path which can be used by the charts for sharing models and datasets.

```yaml
hostCommonPath: /opt/morpheus/common
```

The imagePullPolicy determines whether the container runtime should retrieve an image from a registry to create the pod. Use 'Always' for development.

```yaml
imagePullPolicy: IfNotPresent
```

The SDK pod should run to completion normally and not be restarted. This is done for the non-interactive use case where a user wants the pipeline to run (possibly) indefinitely.

```yaml
restartPolicy: Never
```

Image pull secrets provide the properly formatted credentials for accessing the container images from NGC. It essentially encodes the provided API_KEY. Note that Fleet Command deployments create these secrets automatically based on the FC org, named literally 'imagepullsecret'.

```yaml
imagePullSecrets:
  - name: nvidia-registrykey-secret
# - name: imagepullsecret
```

When deploying to OpenShift we need to create a ServiceAccount for attaching permissions, such as the use of hostPath volumes.

```yaml
serviceAccount:
  create: false
  name: morpheus
```

General flag for OpenShift adjustments.

```yaml
platform:
  openshift: false
```

Use a nodeSelector with OpenShift for GPU affinity. The default is nil for the non GPU Operator/NFD use case. Also, refer to the AI Engine chart deployment.

```yaml
nodeSelector: {}
```
