---
title: "Infrastructure Deployment for Stock recommendation AI Agents and Hyperscalers"
date: "2025-03-26"
categories: AI
layout: post
---

To deploy the infrastructure needed for the [Stock Recommendation AI Agents with OpenShift, OpenShift AI, and Hyperscalers blog](../26/stock-ai-agents), you have two options:

- **Use the Developed Validated Pattern**: This is the recommended approach for a streamlined deployment.
- **Manually Deploy All Dependencies**: This option provides more control but requires more effort.

Both options start from a base OpenShift installation, which can be obtained by following the instructions [here](https://console.redhat.com/OpenShift/install/aws/installer-provisioned).

# How to Install the Validated Pattern

First, clone the validated pattern repository and check out the `rhdh-demo` branch:

```bash
git clone https://github.com/luis5tb/multicloud-gitops/
cd multicloud-gitops
git checkout rhdh-demo 
```

For a deeper understanding of the pattern, refer to the root README file and the one at the developer hub template [here](https://github.com/luis5tb/developerhub-agentic-demo). Pay special attention to the non-default secrets required for installing **Developer Hub Git dependencies**.

### Required Dependencies

These dependencies include:

- **GitHub Application OAuth Secrets**: Needed for setting up Developer Hub in the OpenShift cluster.
- **User's GitHub Account Secrets**: Allow the Developer Hub template to create demo repositories on behalf of the user.
- **Webhook Secrets**: Two secrets are needed to support the creation of webhooks that trigger the building of the AI Agent Service Image in OpenShift whenever the user modifies the AI agent code.
- **Hugging Face Token**: Required when using Hugging Face to download the model to be served, instead of using OCI model storage.
- **VectorDB Providers**: Keys and other information needed to connect to the supported VectorDB hyperscaler providers:
  - Azure AI Search Service
  - AWS Knowledge Base
  - Google Vertex AI Search
  
  Note that the `VECTORDB_PROVIDER` field must be set to `Azure`, `AWS`, or `Google`, depending on your choice.

Once the secrets are configured, it is time to install the pattern:

1. Use the `oc login` command to log in to your OpenShift cluster and go to the *Validated Pattern* cloned repository:
   ```bash
   cd <your base path>/multicloud-gitops
   ```

2. After adding your Git secrets to `values-secret.yaml.template` (following the template in the [README](https://github.com/luis5tb/developerhub-agentic-demo)), copy them to the appropriate folder:
   ```bash
   cp values-secret.yaml.template ~/.config/hybrid-cloud-patterns/values-secret-multicloud-gitops.yaml
   ```

3. Trigger the pattern installation:
   ```bash
   ./pattern.sh make install
   ```

4. **[Optional]** Refresh secrets after the cluster is running in case you need to add or modify any existing ones:
   ```bash
   cp values-secret.yaml.template ~/.config/hybrid-cloud-patterns/values-secret-multicloud-gitops.yaml
   ./pattern.sh make load-secrets
   ```

# OpenShift AI Dependencies

The second option is to install OpenShift AI and its dependencies manually. To install OpenShift AI with the required functionality for the demo, the following components are necessary:

- **OpenShift Serverless** and **OpenShift Service Mesh**: These are required to manage networking and scaling of resources. Note that KServe uses them for ingress management for LLM endpoints.
- **Authorino Operator**: Required for token authentication support for the inference services deploying the LLMs.
- **NVIDIA GPU Operator**: Necessary to handle NVIDIA GPUs.
- **Node Feature Discovery Operator**: Required to discover and annotate the nodes with GPUs.
- **OpenShift Pipelines**: The Developer Hub template creates Tekton pipelines (e.g., to build the agent container image), so it requires this to be available at the OpenShift level.
- **OpenShift DevSpaces**: Installed to provide an integrated experience where the code for the Agent from the Developer Hub template can be modified directly on OpenShift. It is linked on the Developer Hub template.
- **GitWebHook Operator**: Needed by the Developer Hub template to add webhooks to the created code repositories, allowing Tekton pipelines to be automatically triggered upon detected changes.

In addition to the components required by OpenShift AI, there are extra configurations to be made to install and configure OpenShift AI:

### Base OpenShift AI CR

This is standard with the managed services, including the *Model Registry* (configured but not used in the demo):

```yaml
apiVersion: datasciencecluster.opendatahub.io/v1
kind: DataScienceCluster
metadata:
name: default-dsc
spec:
components:
    dashboard:
    managementState: Managed
    workbenches:
    managementState: Managed
    datasciencepipelines:
    managementState: Managed
    kueue:
    managementState: Managed
    codeflare:
    managementState: Managed
    ray:
    managementState: Managed
    modelmeshserving:
    managementState: Managed
    kserve:
    managementState: Managed
    serving:
        ingressGateway:
        certificate:
            type: OpenshiftDefaultIngress
        managementState: Managed
        name: knative-serving
    trustyai:
    managementState: Managed
    modelregistry:
    managementState: Managed
    registriesNamespace: rhoai-model-registries
```

### Accelerator Profile

This is needed to associate GPUs with the workbenches and inference services. More can be added for different types of GPUs:

```yaml
apiVersion: dashboard.opendatahub.io/v1
kind: AcceleratorProfile
metadata:
name: nvidia-gpu
namespace: redhat-ods-applications
annotations:
    argocd.argoproj.io/sync-wave: "1"
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
spec:
displayName: nvidia-gpu
enabled: true
identifier: nvidia.com/gpu
tolerations:
    - effect: NoSchedule
    key: nvidia.com/gpu
    operator: Exists
    value: ''
```

### Model Registry

In addition to the standard **ModelRegistry CR**, it requires an SQL database as a backend, which should also be configured.
