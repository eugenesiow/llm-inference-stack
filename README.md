# **LLM Inference Stack Helm Chart**

This Helm chart deploys a production-ready LLM inference architecture on Kubernetes, utilizing **KServe**, **vLLM**, **Envoy AI Gateway**, **KEDA**, and **Arize Phoenix**.

## **Architecture Overview**

The following diagram illustrates the request flow, control plane scaling, and observability integrations deployed by this chart.

graph TD  
    Client(\[Client / Application\]) \--\> LB\[Load Balancer\]

    subgraph Cluster \[Kubernetes Inference Cluster\]  
        LB \--\> Gateway  
          
        subgraph EAG \[Envoy AI Gateway\]  
            Gateway\[Envoy Gateway Pods\]  
            Feat1\[Rate Limiting\]  
            Feat2\[Auth & Routing\]  
        end  
          
        Gateway \--\> ISVC  
          
        subgraph DataPlane \[Data Plane\]  
            ISVC\["KServe InferenceService\<br\>(vLLM Container)"\]  
        end  
          
        subgraph ControlPlane \[Control Plane\]  
            KEDA\[KEDA Scaler\]  
        end  
          
        KEDA \-.-\>|Scale Replicas| ISVC  
    end

    subgraph Ops \[Observability & Governance\]  
        Vault\[API Key Secret\] \-.-\>|Inject| Gateway  
        Phoenix\["Arize Phoenix\<br\>(Traces/Logs)"\]  
        Prom\["Prometheus\<br\>(Metrics)"\]  
    end

    Gateway \--\>|OTEL Traces| Phoenix  
    ISVC \--\>|Scrape Metrics| Prom  
    Prom \--\>|Query: num\_requests\_waiting| KEDA

1. **Envoy AI Gateway**: The entry point. Handles Authentication (API Keys), Rate Limiting (Token-based), and routing.  
2. **KServe (Data Plane)**: Hosts the InferenceService running the **vLLM** backend.  
3. **KEDA**: Autoscales the KServe pods based on vLLM metrics (e.g., requests waiting).  
4. **Arize Phoenix**: Receives traces and metrics from the Gateway and Model for observability.

## **File Structure**

llm-inference-stack/  
├── Chart.yaml                  \# Helm chart metadata  
├── values.yaml                 \# Default configuration values  
├── README.md                   \# Documentation  
└── templates/  
    ├── configmap-gateway.yaml  \# Configuration for Envoy AI Gateway (routes, rate limits)  
    ├── gateway-deployment.yaml \# Deployment and Service for the AI Gateway  
    ├── inference-service.yaml  \# KServe InferenceService definition (vLLM)  
    └── scaledobject.yaml       \# KEDA ScaledObject for autoscaling logic

## **Prerequisites**

* Kubernetes Cluster (1.24+)  
* **KServe** installed (with Serverless mode/Knative enabled).  
* **KEDA** installed.  
* **Prometheus** (configured to scrape KServe pods).  
* **Arize Phoenix** deployed (for observability).  
* GPU Nodes available (NVIDIA).

## **Installation**

1. Configure values.yaml:  
   Update the model.storageUri to point to your S3 bucket containing the model weights (e.g., Llama-3).  
   model:  
     name: "llama-3-70b"  
     storageUri: "s3://my-org-models/llama-3-70b/"  
     runtime:  
       gpuCount: 2 \# Adjust based on model size

2. **Install the Chart**:  
   helm install llm-stack ./llm-inference-stack \-n kserve-inference

3. **Port Forward (for testing)**:  
   kubectl port-forward svc/llm-stack-ai-gateway 8080:8080 \-n kserve-inference

## **Configuration Details**

### **1\. vLLM & KServe**

The chart creates a v1beta1 InferenceService. It uses a custom container configuration to run vllm.entrypoints.openai.api\_server.

* **Weights**: Must be accessible via the storageUri.  
* **Service Account**: Ensure model.serviceAccountName has IAM permissions to read from your S3 bucket.

### **2\. Autoscaling (KEDA)**

Autoscaling is configured to watch Prometheus metrics.

* **Critical**: Ensure your Prometheus is scraping the KServe pods. vLLM exposes metrics on /metrics.  
* The ScaledObject query in templates/scaledobject.yaml uses vllm:num\_requests\_waiting. You may need to adjust the Prometheus URL in values.yaml to match your cluster's DNS (e.g., prometheus-operated.monitoring...).

### **3\. Envoy AI Gateway**

The gateway is configured via configmap-gateway.yaml.

* **Rate Limiting**: Configured to limit tokens per minute/second.  
* **Routing**: Routes /v1/chat/completions to the KServe internal service.

### **4\. Observability (Phoenix)**

The Gateway is configured to push OTLP traces to the endpoint defined in values.observability.phoenix.endpoint. Ensure this points to your Phoenix collector service.

## **Governance (Vault/Secrets)**

This chart allows mapping an existing Secret (apiKeys.secretName) to the Gateway environment variables.

* **Production**: Use External Secrets Operator to sync Vault secrets to a Kubernetes Secret named ai-gateway-api-keys.  
* **Testing**: Set apiKeys.existingSecret: false and provide a dummyKey in values.