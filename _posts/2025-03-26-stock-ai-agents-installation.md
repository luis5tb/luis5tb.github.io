---
title: "Infrastructure Deployment for Stock recommendation AI Agents and Hyperscalers"
date: "2025-03-26"
categories: AI
layout: post
---

To deploy the infrastructure needed for the [Stock recommendation AI Agents with OpenShift, OpenShift AI and Hyperscalers blog](../26/stock-ai-agents) there are two options:
- Use the developed [Validated Pattern](https://github.com/luis5tb/multicloud-gitops/tree/rhdh-demo).
- Manually deploy all the dependencies.

Both options start from a base OpenShift installation, which can be obtained by following the instructions [here](https://console.redhat.com/OpenShift/install/aws/installer-provisioned).


# How to install the Validated Pattern

First clone the validated pattern repository and checkout the `rhdh-demo` branch.

```bash
git clone https://github.com/luis5tb/multicloud-gitops/
cd multicloud-gitops
git checkout rhdh-demo 
```

For a deeper understanding of the pattern, refer to the root README file, also the one at the developer hub template [here](https://github.com/luis5tb/developerhub-agentic-demo). Pay special attention to the non-default secrets required for installing **Developer Hub Git dependencies**.

These dependencies include:
- **GitHub Application OAuth secrets**: Needed for setting up Developer Hub in the OpenShift cluster.
- **User's GitHub account secrets**: Allow the Developer Hub template to create demo repositories on behalf of the user.
- **Webhook secrets**: Two secrets are needed to support the creation of webhooks that triggers the building of the AI Agent Service Image in OpenShift whenever the user modifies the AI agent code.
- **Huggingface token**: needed when using HuggingFace to download the model to be served, instead of OCI model storage.
- **VectorDB prodivers**: keys and other information needed to connect to the supported vectorDB hyperscalers providers:

  - Azure AI Search Service
  - AWS Knowledge Base
  - Google Vertex AI Search
  
  Note the `VECTORDB_PROVIDER` field that must be set to `Azure`, `AWS` or `Google` depending on the desired one.

Once the secrets are configured it is time to install the pattern:

- Use oc login command to login into your OpenShift cluster
  ```bash
  cd <your base path>/multicloud-gitops
  ```

- After adding your git secrets to `values-secret.yaml.template (following template the [readme](https://github.com/luis5tb/developerhub-agentic-demo)) copy them to the right folder:
  ```bash
  cp values-secret.yaml.template ~/.config/hybrid-cloud-patterns/values-secret-multicloud-gitops.yaml
  ```

- And trigger the pattern installation:
  ```bash
  ./pattern.sh make install
  ```

- [Optional] Refresh secrets after the cluster is running in case you need to add/modify any of the existing ones:
  ```bash
  cp values-secret.yaml.template ~/.config/hybrid-cloud-patterns/values-secret-multicloud-gitops.yaml
  ./pattern.sh make load-secrets
  ```

# OpenShift AI Dependencies

The second option is to install OpenShift AI and its dependencies manually. To install OpenShift AI with the required functionality for the demo the next is required:

- **OpenShift Serverless** and **OpenShift ServiceMesh** to manage networking and scaling of resources. Note KServe uses them for ingress management for LLM endpoints.
- **Authorino Operator**: Required for token authentication support for the inference services deploying the LLMs.
- **NVIDIA GPU Operator**: Required to handle the NVIDIA GPUs.
- **Node Feature Discovery Operator**: Required to discover and annotate the nodes with GPUs.
- **OpenShift Pipelines**: The Developer Hub template creates Tekton pipelines (e.g., to build the agent container image), so it requires it to have it available at the OpenShift level.
- **OpenShift DevSpaces**: Installed to have an integrated experience where the code for the Agent from the Developer Hub template can be modified directly on OpenShift. It is linked on the Developer Hub template.
- **GitWebHook Operator**: Needed by the Developer Hub template to add webhooks to the created code repositories. Then Tekton pipelines can be automatically triggered upon changes detected.

In addition to the components required by OpenShift AI, there are extra configurations to be made to install and configure  OpenShift AI:
- **Base OpenShift AI CR**: Standard with the managed services, with the inclusion of *Model Registry* (configured but not used in the demo)
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

- **Accelerator Profile**: Needed to associate GPUs to the workbenches and inference services. More can be added in case of different type of GPUs:
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

- **Model Registry**: Besides the standard **ModelRegistry CR**, it requires an SQL DataBase as a backend, so one should be configured too.
