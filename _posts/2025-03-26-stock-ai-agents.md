---
title: "Stock Recommendation AI Agents with OpenShift, OpenShift AI, and Hyperscalers"
date: "2025-03-26"
categories: AI
layout: post
---

#### Authors
- Sharon Dashet
- Luis Tomás Bolívar

### Preface

As we transition from initial Generative AI proofs of concept (POCs) and chatbots to production-ready AI systems, [AI Agents and Agentic AI](https://www.youtube.com/watch?v=KrRD7r7y7NY) are emerging as the driving force behind intelligent automation in enterprises. These agents, capable of collaborating, understanding complex tasks, retrieving information, and making informed decisions, are transforming how teams operate.

In this demo, we will explore how to build and fully operationalize AI Agents using Red Hat OpenShift, Red Hat Developer Hub, and OpenShift AI, focusing on a Stock Recommendation use case. We will also demonstrate how to evaluate and safeguard AI systems at scale using technologies such as [Trusty AI](https://github.com/trustyai-explainability) and managed cloud services.

Responsible AI is a central theme in our demo, as we require the AI Agent to adhere to safety and regulatory standards. Additionally, we aim to evaluate and observe model performance and accuracy at all stages of the AI system, which is crucial in highly regulated industries such as fintech.

Furthermore, we will demonstrate how these agents can seamlessly utilize Generative AI services from public cloud and SaaS platforms, including Amazon's AWS Bedrock, Microsoft Azure AI Foundry, and Google Vertex AI, through integrations offered by Agentic frameworks like LangGraph.

Throughout this blog, we will use the example of a fictitious large corporation's fintech team responsible for investment and portfolio management. After several successful POCs and early initiatives with Generative AI chatbots, the team is now exploring a more autonomous approach that leverages the latest advancements in AI systems. Their goal is to build an AI system that can automate parts of their stock recommendation process with minimal user guidance.

## Stock Recommendation AI Agents

The team's initial high-level architecture for the AI system is depicted below. During the demo walkthrough, we will break it down into its logical components.

![The Stock Recommendation Agent Application Simplified Architecture](../../../../images/stock-ai-agents/simplified_architecture.png?w=1024)

The architecture builds on the team's existing **hybrid multi-cloud** investments, leveraging their application development, platform engineering, AI, and data platforms.

The AI Agent service is intended to be deployed in their OpenShift production cluster using a fully automated **GitOps approach**.

Meta's LLM, [Llama-3.2](https://huggingface.co/meta-llama/Llama-3.2-3B-Instruct), will be deployed on their OpenShift AI platform, utilizing the same OpenShift production cluster. The model is expected to leverage existing investments in OpenShift NVIDIA GPU worker nodes. To balance cost and performance, the team plans to select an open-source SLM (small language model) that performs well in their evaluations and benchmarking. Given their hybrid environments, a local open-source model that can be efficiently served on their GPUs is an ideal fit.

To ground their AI Agent with relevant financial context (e.g., the financial reports of public companies), they plan to use **RAG (Retrieval-Augmented Generation)**. The company's data platform services are distributed across multiple public cloud platforms, and it has been decided that the AI Agent will integrate with these services.

For RAG, the team has been using [AWS Bedrock Knowledge Bases](https://aws.amazon.com/bedrock/knowledge-bases/), a managed RAG service that includes a managed Vector DB, an offline pipeline to ingest financial data into the vector DB, and an easy retrieval API.

They also plan to utilize web search for real-time stock data context and other tools for interacting with the environment and performing actions as part of their agentic workflow. To that end, their model of choice should have built-in support for [tool calling](https://docs.vllm.ai/en/latest/features/tool_calling.html).

Ensuring **Responsible AI** is a **top priority** for the team. To assess the model's performance against their goals—specifically in terms of accuracy, relevance, and coherence—they plan to implement offline model evaluations based on open-source projects like [lm evaluation harness](https://github.com/EleutherAI/lm-evaluation-harness). Additionally, to safeguard the AI Agent during runtime, they intend to incorporate AI safety measures through the use of guardrails—policies and constraints designed to prevent harmful or unintended outputs based on guard models like [granite guardian](https://huggingface.co/ibm-granite/granite-guardian-3.0-8b).

The stock recommendation use case in our demo is heavily inspired by the MarketMaestro blog series ([here](https://medium.com/@vcaldeir/marketmaestro-building-and-aligning-a-local-ai-stock-advisor-agent-with-instructlab-podman-ai-lab-a0aa0041e843) and [here](https://medium.com/@vcaldeir/marketmaestro-building-and-aligning-a-local-ai-stock-advisor-agent-with-instructlab-podman-ai-lab-ec0be37f3f49)), showcasing how to align, evaluate, and deploy an AI Stock Advisor Agent in a local environment with InstructLab, Podman AI Lab, and LangChain.

## Demo Setup

### OpenShift Prerequisites

At a minimum, an OpenShift cluster (version 2.17 or newer) should be installed on-premises or in the cloud.
- The worker nodes should have a GPU that supports the inference model you plan to use for the demo. 
We have tested the demo with the default model: "Llama-3.2-3b-Instruct" and NVIDIA A10G Tensor Core GPUs (instance type `g5.4xlarge`).

The demo also requires the following key OpenShift applications to be installed:
- **Developer Hub** and **the AI template** – The AI template provisions the demo resources in the user's namespaces and Git account.
- **OpenShift GitOps** (based on the upstream project Argo CD) – Used to operationalize the AI Agent.
- **OpenShift AI** – Hosts the LLM model and includes necessary networking, authorization, and GPU dependencies. It also integrates the Trusty AI operator for evaluating the AI system.
- **[Optional] MinIO** – Facilitates downloading model files to OpenShift AI. It is one of the available methods for obtaining model files needed for serving the model in OpenShift AI, alongside using an OCI registry URI.

To satisfy these extensive requirements, we created an automation in the form of a "Red Hat validated pattern." The validated pattern can easily be applied to your OpenShift cluster and take care of all these requirements. Follow these detailed instructions to install the pattern: [Infrastructure Deployment for Stock Recommendation AI Agents and Hyperscalers](../26/stock-ai-agents-installation).

Note that each of these applications has lower-level dependencies (e.g., Autorino, NFD, Service Mesh, etc.), which are automatically handled by the validated pattern setup. If you prefer to set up these components manually, you can refer to the exact requirements in this [post](../26/stock-ai-agents-installation).

Once all prerequisites are met, you can open Developer Hub in your OpenShift demo cluster and use the "**Stock Recommendation AI Agents with OpenShift, OpenShift AI, and Bedrock**" AI template to provision the demo resources.

### Unpacking the Demo Developer Hub AI Template

To set up the demo, we use a **Scaffold Software Template** on **Red Hat Developer Hub** (built on the upstream **IDP project Backstage**). Using a software template on Developer Hub offers repeatability and can be used to install the demo with multiple configurations and in multiple tenants sharing the same cluster.

#### Importing the Template to Developer Hub

If you have used the validated pattern to install the OpenShift prerequisites, then the template will be installed for you in Developer Hub, and no further action is required.

If you can satisfy the demo requirements yourself without the validated pattern, you can import the template to Developer Hub directly from [here](https://github.com/luis5tb/developerhub-agentic-demo). If you would like to modify the template, you can fork the repository and import your fork.

#### Provisioning the Demo AI Agent Resources Using the Template

Log in to Developer Hub and locate the template by navigating to the Create side menu. Look for a template named: **"Stock Recommendation AI Agents with OpenShift, OpenShift AI, and Bedrock"**, click choose, and follow the setup screens.

![RHDH Template](../../../../images/stock-ai-agents/rhdh_template.png?w=1024)

You will need to provide the following mandatory inputs during the template setup:
- Information about GitHub location and component description:
  - **GitHub Organization**: your GitHub User that was configured for pushing repositories to your GitHub account.
  - [Optional] **Description**: provide an optional description for the component.
- Component deployment information:
  - **Cluster ID**: OpenShift apps route.
  - **Namespace**: target namespace, for example: `fintech-team`.
  - **ArgoCD Instance**: Set to `default` by the Validated Pattern. If manually deployed, adapt to your configured instance in RHDH.
  - **ArgoCD Namespace**: Set to `multicloud-gitops-hub` by the Validated Pattern. If manually deployed, adapt it to the one you configured.
  ![Components information](../../../../images/stock-ai-agents/rhdh_components_information.png?w=1024)
- Model Information:
  - **Model Name**: Name for the `ServingRuntime`.
  - **ServingRuntime Tool Parser**: Required for VLLM ServingRuntime configuration with tool calling.
  - **Model Location**: Choose between S3 and OCI options:
    - **S3**:
      - **HuggingFace Model Path**: path to the model on Hugging Face.
      - **S3 endpoint** (deployed MinIO) and **S3 bucket** to store the downloaded model.
      - **Vault Setup**: Ensure the `huggingface-keys` vault exists (with an `hf_token` key) at the Vault Storage. The template retrieves this token to authenticate with Hugging Face and download accessible models.
    - **OCI**:
      - **OCI URI**: The model container image URI, starting with `oci://`.
      - [Optional] **Registry secret**: Name of the secret containing Registry credentials if needed for fetching the model. The secret can be created using:
        ```bash
        $ oc create secret -n vllm-fintech-team docker-registry ltomasbo-quay \
            --docker-server=quay.io \
            --docker-username=XXXX \
            --docker-password=XXXX \
            --docker-email=XXXX
        ```

![Model Information (OCI option)](../../../../images/stock-ai-agents/rhdh_oci.png?w=1024)

- Agent Image build information:
  - **Image Host**: The registry to use for agents container image builds.
  - **Image Tag**: Tag for the agent container image being built. The image name is generated based on the `namespace name` + `agent-hyperscalers-app`.
- **Review & Deploy**: Check inputs and trigger creation.

![Template deployment](../../../../images/stock-ai-agents/rhdh_template_deploy.png?w=1024)

After the template run is completed, OpenShift GitOps tooling will deploy all your demo resources (see Section **Red Hat Developer Hub Template Insights** for more information). This asynchronous deployment method is facilitated by the GitOps Git Repository created by the template, which defines the ArgoCD applications to be deployed. Its status can be tracked on the RHDH catalog.

![RHDH Catalog (Components)](../../../../images/stock-ai-agents/rhdh_catalog.png?w=1024)

The deployments may take 15+ minutes. Track deployment progress using the OpenShift Argo CD Hub application. If you are not using the Validated Pattern, then select the Argo CD instance used by the template instantiation. Once all the resources are successfully deployed by Argo CD, you can start experimenting with the demo. Go ahead and jump right into the **Demo Walkthrough** Section to see our AI Agent in action.

To explore in detail what is behind the template, check Section **Red Hat Developer Hub Template Insights**.

## Demo Walkthrough

Once the template deployment is complete, you can:
- **Evaluate the LLM** – Use **TrustyAI Evaluation Jobs** to assess the deployed language model.
- **Interact with AI Agents** – Access the **Gradio server** to start engaging with the deployed agents.
- **Modify & Redeploy** – Update the agents' code and trigger a redeployment to test changes in real-time.

### Run the Trusty AI Evaluation Job

Since the LLM is at the core of the agents, evaluating its performance before deployment is essential to ensure **accuracy, relevance, and coherence** for the intended tasks. Pre-deployment evaluation helps validate that the selected model aligns with the specific use case, minimizing the risk of agent misbehavior.

To streamline this process, we leverage **TrustyAI**, included in the Validated Pattern as part of the OpenShift AI installation. It is instantiated via the Developer Hub template, which defines a **Pipeline** that, upon execution, generates an `LMEvalJob` object. The **TrustyAI service** detects this object and initiates a **Kubernetes Job** that triggers the LLM evaluation using the **open-source** [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness) framework. For more details, see Section **Red Hat Developer Hub Template Insights**.

This **automated evaluation** ensures that the selected model performs reliably under expected workloads.

#### Triggering the LLM Evaluation Pipeline (Manual Process)

To execute the **LLM evaluation** using the Pipeline created by the Developer Hub template, follow these steps:
1. Navigate to the `vllm-fintech-team` namespace.
2. Manually trigger the evaluation Pipeline, providing the required parameters.
![Run Evaluation Pipeline](../../../../images/stock-ai-agents/pipeline_run.png?w=1024)
![Pipeline Parameters](../../../../images/stock-ai-agents/pipeline_parameters.png?w=1024)
3. Select the evaluation method:
    * Use a predefined `TaskList` (set of tasks/subtasks).
    * Use a custom `TaskRecipe` for a tailored evaluation.
4. The Pipeline creates an `LMEvalJob` **Custom Resource (CR)** with the selected parameters.
5. The **TrustyAI service** detects the `LMEvalJob` CR and generates a new Job to run the evaluation.
6. Monitor the results in:
    * The `Job` Logs.
    * The `LMEvalJob` CR status.

#### Beyond Pre-Deployment: Adding Runtime Guardrails

In addition to pre-deployment evaluations, we plan to introduce **runtime guardrails** to act as a **safety gateway** between the agents and the LLM. These guardrails will be inspired by those defined by **Hyperscalers**.

### Using the AI Agent for Stock Recommendation and Analysis

In the Developer Hub Catalog, you can check the topology of the deployed template and view the deployed Agent. Click on the deployed agent to interact with it.

![RHDH Component Topology: Agent App](../../../../images/stock-ai-agents/rhdh_topology.png?w=1024)
![Gradio ChatBot](../../../../images/stock-ai-agents/gradio.png?w=1024)

Even though this appears as a single Pod in OpenShift, it consists of multiple **specialized agents** working together. The architecture is as follows:
- **Gradio** – Provides a simple user interface for interaction.
- **LangGraph** – Orchestrates the multi-agent workflow.
- The deployed system consists of three specialized agents working together to analyze stocks, all leveraging the LLM hosted in OpenShift AI with the template:
  1. **Search Agent** - Gathers real-time stock information from the internet using **DuckDuckGo**. It follows a **ReAct** pattern and can make several calls to the tool until it retrieves all relevant information.
  2. **Summary Agent** - Utilizes the real-time search data passed by the orchestrator from the Search Agent and performs a similarity search using **AWS Bedrock Knowledge Base** to access internal documents. It then sends the augmented prompt to the LLM to summarize the stock.
  3. **Recommendation Agent** - Uses the summary passed by the orchestrator (generated by the Summary Agent) to perform a final analysis and provide a recommendation for the stock, along with the reasoning.

![The Stock Recommendation Agent Application Logical Workflow](../../../../images/stock-ai-agents/logical_workflow.png?w=1024)

#### Using Bedrock Knowledge Base - Managed RAG Service on AWS

The **Bedrock Knowledge Base** is a managed **RAG** (Retrieval-Augmented Generation) service on AWS, used to store and retrieve information from a knowledge base. The **Summary Agent** leverages this service to access the knowledge base and retrieve relevant information for summarization.

Here's how you can create and configure a new Knowledge Base in AWS Bedrock:
1. **Go to AWS Bedrock's** `Builder tools` > `Knowledge Bases`.
2. **Create a new Knowledge Base**: `Knowledge Base with vector store`.

![AWS Knowledge Bases](../../../../images/stock-ai-agents/aws_knowledgebase.png?w=1024)

3. **Select Data Source**: Choose the desired Data Source, e.g., Amazon S3.

![Data Sources](../../../../images/stock-ai-agents/aws_data_sources.png?w=1024)

4. **Upload documents**: Pre-create a new S3 bucket for the Knowledge Base and upload the documents you want to use. Then select them in the form. For this demo, we added a couple of files with fake data about two companies. You can see the data uploaded [here](https://github.com/luis5tb/developerhub-agentic-demo/tree/main/skeleton/fake_reports). The agent's reply includes the data from the uploaded files.

![Fake Reports](../../../../images/stock-ai-agents/aws_fake_reports.png?w=1024)

5. **Create Knowledge Base**: Once the form is filled, click on `Create` and wait for the Knowledge Base to be created.
6. **Retrieve Knowledge Base ID**: Get the Knowledge Base ID, which is required to configure the secret.

To use the Bedrock Knowledge Base in your agent/application, you must configure the necessary **credentials** in the secret. For the Validated Pattern, configure the credentials in the `values-secret.yaml.template` file with the correct AWS Knowledge Base credentials and reload the secrets:

```yaml
- name: vectordb-keys
  fields:
  - name: VECTORDB_PROVIDER
    value: AWS
  - name: AWS_KNOWLEDGE_BASE_ID
    value: XXXX
  - name: AWS_REGION_NAME
    value: XXXX
  - name: AWS_ACCESS_KEY_ID
    value: XXXX
  - name: AWS_SECRET_ACCESS_KEY
    value: XXXX
```

#### Azure AI Search Configuration

Similarly, **Azure AI Search Service** is also supported. To use it, change the `VECTORDB_PROVIDER` environment variable to `AZURE` and set the correct Azure AI Search Service credentials. For the Validated Pattern, update the `values-secret.yaml.template` file with the correct Azure AI Search Service credentials and reload the secrets:

```yaml
- name: vectordb-keys
  fields:
  - name: VECTORDB_PROVIDER
    value: AZURE
  - name: AZURE_AI_SEARCH_SERVICE_NAME
    value: XXXX
  - name: AZURE_AI_SEARCH_API_KEY
    value: XXXX
  - name: AZURE_AI_INDEX_NAME
    value: XXXX
```

#### Google Vertex AI Search Configuration

Similarly, **Google Vertex AI Search Service** is also supported. To use it, change the `VECTORDB_PROVIDER` environment variable to `GOOGLE` and set the correct Vertex AI Search Service credentials. For the Validated Pattern, update the `values-secret.yaml.template` file with the correct Vertex AI Search Service credentials and reload the secrets:

```yaml
- name: vectordb-keys
  fields:
  - name: VECTORDB_PROVIDER
    value: GOOGLE
  - name: GOOGLE_API_KEY
    value: YOUR_GOOGLE_API_KEY.json
  - name: GOOGLE_PROJECT_ID
    value: XXX
  - name: GOOGLE_LOCATION_ID
    value: "global"
  - name: GOOGLE_DATA_STORE_ID
    value: XXXX
```

Note that you will need to download your **service account** `API KEY` in JSON format and point the `GOOGLE_API_KEY` to the file.

The steps to create the data store and make it available are as follows:
1. Create a **bucket** in **Cloud Storage** and upload the desired data.
2. In the **Agent Builder**, create a **Data Store**, selecting the data source, in this case, Cloud Storage.
   * Point the **Data Store** to the Cloud Storage data.
3. In the **Agent Builder**, create an **App** pointing to the created **Data Store**.
   * Enable Enterprise edition in the **Advanced Configuration** tab to allow retrieval.

## Modifying the AI Agent Application

You can update the **LangGraph Agent(s) code** using two options available in **Red Hat Developer Hub**:
- **OpenShift Dev Spaces**: Click on the link to OpenShift Dev Spaces (VS Code) for a web-based VS Code environment where you can directly edit the code and push the changes to the associated GitHub repository.
- **GitHub**: Use the `View Source` link to directly edit code in your GitHub repository (the one created for the Agent code by the Developer Hub template).

![RHDH Component Links](../../../../images/stock-ai-agents/rhdh_links.png?w=1024)

With both options, **push the updated code** to the Agent's code repository. This automatically triggers a **PipelineRun**, which:
1. **Builds** a new Agent Service image (see the next Section for more details).
2. **Deploys** the updated version, rolling out the new image.

![OpenShift PipelineRun: Agent Container Image Build](../../../../images/stock-ai-agents/pipeline_build.png?w=1024)

### Red Hat Developer Hub Template Insights

The template deploys resources in OpenShift under two different OpenShift namespaces:
- One for the ML backend resources, primarily owned by OpenShift AI.
- One for the AI application side resources atop OpenShift.

The two namespaces are based on the `Namespace` input variable provided during the Developer Hub template run. In our example, we use `fintech-team` as the namespace parameter. These two namespaces are created: `vllm-fintech-team` and `fintech-team`.

#### What is Inside the Namespaces?

##### vllm-fintech-team

Namespace `vllm-fintech-team` contains these ML resources in OpenShift and OpenShift AI:
- **vLLM Inference Service** with the Gen AI model deployed in OpenShift AI. The model is determined by the model input variables. The default is `granite`, but we used `llama-3.2`.
- An **OpenShift Tekton Pipeline** called `evaljob-pipeline` to run a TrustyAI evaluation job against the LLM service in OpenShift AI. The pipeline is not triggered by default.
- The **TrustyAI service**.

Here is a simplified depiction of the `vllm-fintech-team` namespace:

![vllm-fintech-team namespace](../../../../images/stock-ai-agents/vllm-fintech-team-namespace.png?w=1024)

##### fintech-team

Namespace `fintech-team` contains these application resources in OpenShift:
- An **OpenShift Tekton Pipeline** to build the AI Agent application image (called `agent-build-pipeline` and `agent-build-pipelinerun`).
- The **AI Agent application** deployed under the `agent` service using the latest image built by the pipeline.

Here is a simplified depiction of the `fintech-team` namespace:

![fintech-team namespace](../../../../images/stock-ai-agents/fintech-team-namespace.png?w=1024)

#### Git Repositories

The template creates these two Git repositories based on the `Namespace` input variable:
- `<Namespace input variable>-agent-hyperscalers`
- `<Namespace input variable>-agent-hyperscalers-gitops`

The Git account used is based on the `GitHub Organization` input variable provided during the Developer Hub template run.

In our example, the template provisions these two Git repositories:
- `https://github.com/<the team's git account>/fintech-team-agent-hyperscalers`:
This repo contains the **agent Python code** that is pulled by the Tekton Pipeline to build the Agent image. This means that even after the demo is deployed, you can evolve and edit the project directly on your Git account. The template also installs a **webhook** to track commits and trigger the application build Pipeline in OpenShift automatically upon code changes to the agent repository. This way, you can keep your AI Agent service updated with the latest code changes.
- `https://github.com/<the team's git account>/fintech-team-hyperscalers-gitops`:
**Helm charts** and **ArgoCD applications** used by ArgoCD in the destination OpenShift cluster to automatically sync and deploy the demo resources in the two namespaces mentioned above.

#### Secrets

The template also utilizes secret management in OpenShift (facilitated by [Vault](https://developer.hashicorp.com/vault/docs/platform/k8s/vso) Kubernetes operator) to retrieve the secrets that were previously created and persisted by the validated pattern. If you did not use the Validated Pattern to install the OpenShift demo dependencies, you can create and persist these secrets yourself in OpenShift using Vault or another secret manager.

The following is a list of needed secrets in the Vault:
- **Environment Setup**:
  - **minio-secret**: user and password for the MinIO deployment.
  - **rhdh-keys**: **GitHub OAuth client ID** and **access token**, and the **GitHub Token** so that users can authenticate in RHDH and the template can create the repositories.
  - **modelregistry-keys**: values for the database configuration that the model registry uses.
- **RHDH Template**:
  - **github-keys**: GitHub secrets so that the template can automate the WebHook creation for the agent codebase Git repository.
  - **vectordb-keys**: configuration and credentials for the VectorDBs usage (AWS Bedrock Knowledge Base, Azure AI Search Service, or Google Vertex AI Search).
  - [optional] **huggingface-keys**: HuggingFace authentication `token`. Needed when S3 is used to serve the model, as some models require authentication to be downloaded from HuggingFace—e.g., Llama models.

Note that the Vault is also being used by the template to copy secrets between namespaces, such as to get access to the Inference Service Token from the Agent namespace. If you use another secret store, the secretStore can be configured in the templates by adapting the `values.yaml` in the helm templates, for example [here](https://github.com/luis5tb/developerhub-agentic-demo/blob/2f31969ff116544ce137ba624b566209fbd79217/manifests/helm/agent/values.yaml#L42).

In addition, after the template deployment is triggered, another secret is required in the VLLM namespace (not in the Vault):
- **registry-keys**: The `.dockerconfigjson` with the credentials to fetch images (ModelCar containers) from the registry. This is required when OCI is used to serve the models instead of S3. The name is passed on the template, and it can be created with:
    ```bash
    $ oc create secret -n VLL_NAMESPACE docker-registry SECRET_NAME \
        --docker-server=REGISTRY_NAME \
        --docker-username=USERNAME \
        --docker-password=PASSWORD \
        --docker-email=EMAIL
    ```

## What is Next

### Llama Stack and MCP

- For the Agentic framework, we plan to complement LangGraph with [LlamaStack](https://github.com/meta-llama/llama-stack)—a collection of standard and interoperable APIs designed to support the core building blocks required to bring generative AI applications to market (think of it as the Gen AI middleware APIs). These building blocks are delivered as interoperable APIs, enabling a wide range of Service Providers to offer their own implementations. This approach allows AI developers to switch seamlessly between compute environments and AI and data platform providers (such as Red Hat AI, AWS Bedrock, Azure, Databricks, and Ollama) with minimal adjustments to configurations. As one example, with a Red Hat AI distribution of the Llama Stack (Llama Stack HTTP server on OpenShift) the agent application will leverage standard Llama Stack Inference API calls with the remote [vLLM Provider](https://blog.vllm.ai/2025/01/27/intro-to-llama-stack-with-vllm.html) instead of relying on proprietary APIs like OpenAI.
  - As part of the Llama Stack release, we will also move the tools execution to an [MCP Server](https://docs.anthropic.com/en/docs/agents-and-tools/mcp). MCP is an open-source protocol that functions as a universal standard for AI systems to access external data sources and tools through a client-server architecture. [Llama Stack MCP provider](https://github.com/meta-llama/llama-stack/tree/main/llama_stack/providers/remote/tool_runtime/model_context_protocol) will act as the MCP client connecting the Agentic AI services of the demo to tools and data sources. MCP was developed and open sourced by Anthropic but quickly got adopted by Industry leaders like MS and OpenAI. With more MCP servers, clients and integrations it is becoming the De facto protocol to bridge between agentic systems and the enterprise digital environment.

### Responsible AI : Safety (Guardrails)

- We plan to complement the evaluation part of the Responsible AI with guardrails protecting the runtime application. Options are Trusty AI guardrails with a Granite guard model or Bedrock/Azure Foundry managed guardrails service. This part will ideally be done using the [Llama Stack Safety API](https://github.com/meta-llama/llama-stack/tree/main/llama_stack/providers/remote/safety).


### Known Issues
#### Developer Hub GitHub Authentication 
The recommended way to configure Developer Hub GitHub authentication is with an organization Git account, as explained [here](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.3/html/authentication/authenticating-with-github). This allows multiple users to use the same Developer Hub instance.

For simplicity, the Validated Pattern allows you to set up Developer Hub with personal GitHub authentication. If you took this approach, please note that only one Git user can use the demo parts on Developer Hub.

#### Validated Pattern Repository
Currently, the pattern is located in one of the demo's repositories: https://github.com/luis5tb/multicloud-gitops/tree/rhdh-demo. We plan to move it to the formal Validated Patterns repository.

#### Agent Image Building Tekton PipelineRun Failing
When running the AI template, sometimes the `PipelineRun` (responsible for triggering the image build pipeline of the AI Agent) fails due to a race condition in Argo CD.

To overcome this, remove the failed `PipelineRun` from OpenShift; Argo CD will recreate it, and both the image build and the AI Agent deployment will work as expected.

#### Allow TrustyAI to Download from HF
By default, TrustyAI blocks remote code execution. There is no supported way of configuring this through the operator, so a manual workaround is needed by editing the ConfigMap `trustyai-service-operator-config` in the `redhat-ods-applications` namespace:
- Add the following annotation and save it: `opendatahub.io/managed: 'false'`
- Change `lmes-allow-online` and `lmes-allow-code-execution` to true and save it.
```bash
$ oc annotate configmap trustyai-service-operator-config -n redhat-ods-applications  opendatahub.io/managed=false
$ oc patch configmap trustyai-service-operator-config -n redhat-ods-applications --type merge -p '{"data":{"lmes-allow-online":"true","lmes-allow-code-execution":"true"}}'
```

#### Manual Secret Creation Needed for Private ModelCar OCI Models
When deploying the model from a container registry, you need to pass the OCI URI in the Red Hat DeveloperHub template and optionally a pointer to a secret containing the credentials to pull the model image from that registry. As the VLLM namespace is created during the template deployment phase, creating that secret is left as a manual action after instantiating the template. This allows easy configuration of different registries for different models.
