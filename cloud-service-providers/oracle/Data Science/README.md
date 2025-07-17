# NIM Model Deployment on OCI Data Science
This guide provides step-by-step instructions for deploying **NVIDIA NIM (NVIDIA Inference Microservices)** models on **OCI Data Science**. Example: deploying the **LLaMA3 8B** model. The same process applies to other NIM models.

## ✅ Prerequisites
- OCI CLI installed and configured (`oci setup config`)  
- OCI Data Science service enabled in your tenancy  
- NGC API Key (to pull NIM containers/models)  
- OCI Container Registry (OCIR) access  
- (Optional) OCI Vault for securely storing secrets  

## 1. Setup Configuration
Create a `config.sh` file:
```bash
export TENANCY_OCID="<your-tenancy-ocid>"
export COMPARTMENT_OCID="<your-compartment-ocid>"
export REGION="<your-region>"
export PROJECT_NAME="nim-llama3-8b"
export OCIR_NAMESPACE="<your-tenancy-namespace>"
export OCIR_REPO="${REGION}.ocir.io/${OCIR_NAMESPACE}/nim"
export NGC_API_KEY="<your-ngc-api-key>"
export MODEL_NAME="meta/llama3-8b-instruct"
```
Load it:
```bash
source ./config.sh
```

## 2. Authenticate with OCI
Verify you are authenticated and retrieve your OCIR namespace:
```bash
oci os ns get
```

## 3. Create OCI Data Science Project
Create the project:
```bash
oci data-science project create \
  --compartment-id $COMPARTMENT_OCID \
  --display-name $PROJECT_NAME
```
Export the Project OCID:
```bash
export PROJECT_OCID="<returned-project-ocid>"
```
(Optional) Launch a notebook session:
```bash
oci data-science notebook-session create \
  --compartment-id $COMPARTMENT_OCID \
  --project-id $PROJECT_OCID \
  --display-name "${PROJECT_NAME}-notebook" \
  --notebook-session-configuration-details '{"shape":"VM.GPU.A10.1","blockStorageSizeInGBs":200}'
```

## 4. Store NGC API Key
### Option 1: OCI Vault (Recommended)
```bash
oci vault secret create-base64 \
  --compartment-id $COMPARTMENT_OCID \
  --vault-id <vault-ocid> \
  --secret-name "NGC_API_KEY" \
  --secret-content-content "$(echo -n $NGC_API_KEY | base64)"
```
### Option 2: Environment Variable (Simpler)
Pass `$NGC_API_KEY` directly during deployment.

## 5. Push NIM Container to OCIR
```bash
docker login ${REGION}.ocir.io
docker login nvcr.io
docker pull nvcr.io/nim/meta/llama3-8b-instruct:latest
docker tag nvcr.io/nim/meta/llama3-8b-instruct:latest ${OCIR_REPO}/llama3-8b-instruct:latest
docker push ${OCIR_REPO}/llama3-8b-instruct:latest
```

## 6. Create Model Deployment
```bash
oci data-science model-deployment create \
  --compartment-id $COMPARTMENT_OCID \
  --project-id $PROJECT_OCID \
  --display-name "llama3-8b-nim-deployment" \
  --model-deployment-configuration-details "{\"modelDeploymentInstanceShapeName\":\"VM.GPU.A10.1\",\"replicaCount\":1,\"bandwidthMbps\":100,\"containerConfigDetails\":{\"image\":\"${OCIR_REPO}/llama3-8b-instruct:latest\",\"env\":[{\"name\":\"NGC_API_KEY\",\"value\":\"${NGC_API_KEY}\"},{\"name\":\"MODEL_NAME\",\"value\":\"${MODEL_NAME}\"}]}}"
```
Export the endpoint:
```bash
export ENDPOINT_URL="<returned-endpoint-url>"
```

## 7. Verify Deployment
```bash
curl -X POST "$ENDPOINT_URL/v1/chat/completions" \
  -H "accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
  "messages": [
    {
      "role": "system",
      "content": "You are a polite and respectful chatbot helping people plan a vacation."
    },
    {
      "role": "user",
      "content": "What should I do for a 4 day vacation in Spain?"
    }
  ],
  "model": "meta/llama3-8b-instruct",
  "max_tokens": 16,
  "top_p": 1,
  "n": 1,
  "stream": false,
  "stop": "\n",
  "frequency_penalty": 0.0
}'
```

✅ **You have now deployed a NIM model on OCI Data Science!**
