# Makefile for RAG Deployment
# Replaces the original deploy.sh script with additional uninstall functionality

ifeq ($(NAMESPACE),)
ifeq (,$(filter list-models% help,$(MAKECMDGOALS)))
$(error NAMESPACE is not set)
endif
endif

# Default values
POSTGRES_USER ?= postgres
POSTGRES_PASSWORD ?= rag_password
POSTGRES_DBNAME ?= rag_blueprint
MINIO_USER ?= minio_rag_user
MINIO_PASSWORD ?= minio_rag_password
RELEASE_NAME ?= rag
CHART_PATH ?= rag-ui
HF_TOKEN ?= $(shell bash -c 'read -r -p "Enter Hugging Face Token: " HF_TOKEN; echo $$HF_TOKEN')
DT_API_TOKEN ?= $(shell bash -c 'read -r -p "Enter Dynatrace API Token: " DT_API_TOKEN; echo $$DT_API_TOKEN')
LLM_SERVICE_CHART_PATH ?= llm-service
LLM_SERVICE_RELEASE_NAME ?= llm-service
MCP_SERVERS_CHART_PATH ?= mcp-servers
MCP_SERVERS_RELEASE_NAME ?= mcp-servers
INGESTION_PIPELINE_CHART_PATH ?= ingestion-pipeline
INGESTION_PIPELINE_RELEASE_NAME ?= ingestion-pipeline
CONFIGURE_PIPELINE_SERVER_CHART_PATH ?= configure-pipeline-server
CONFIGURE_PIPELINE_SERVER_RELEASE_NAME ?= configure-pipeline-server
TOLERATIONS_TEMPLATE=[{"key":"$(1)","effect":"NoSchedule","operator":"Exists"}]
OTEL_COLLECTOR_CHART_PATH ?= otel-collector
OTEL_COLLECTOR_RELEASE_NAME ?= otel-collector


#ingestion pipeline configuration
SOURCE ?= S3
EMBEDDING_MODEL ?= all-MiniLM-L6-v2
INGESTION_PIPELINE_NAME ?= demo-rag-vector-db
INGESTION_PIPELINE_VERSION ?= 1.0
ACCESS_KEY_ID ?= $(MINIO_USER)
SECRET_ACCESS_KEY ?= $(MINIO_PASSWORD)
BUCKET_NAME ?= documents
ENDPOINT_URL ?= http://minio:9000
REGION ?= us-east-1
# PDF file path variable for upload-pdf target
PDF_FILE = ../../notebooks/Zippity_Zoo.pdf

S3_TEMPLATE={"access_key_id":"$(1)","secret_access_key":"$(2)","bucket_name":"$(3)","endpoint_url":"$(4)","region":"$(5)"}


# Default target
.PHONY: help
help:
	@echo "Available targets:"
	@echo "  install       - Install the RAG deployment (creates namespace, secrets, and deploys Helm chart)"
	@echo "  install-cpu   - Install the RAG deployment without GPU (creates namespace, secrets, and deploys Helm chart)"
	@echo "  uninstall     - Uninstall the RAG deployment and clean up resources"
	@echo "  status        - Check status of the deployment"
	@echo "  wait          - Wait for all pods to be ready and verify deployment health"
	@echo "  list-models       - List available models for GPU"
	@echo "  list-models-cpu   - List available models for CPU"
	@echo ""
	@echo "Configuration options (set via environment variables or make arguments):"
	@echo "  NAMESPACE                - Target namespace (default: llama-stack-rag)"
	@echo "  RELEASE_NAME             - Helm release name (default: rag)"
	@echo "  CHART_PATH               - Path to the Helm chart (default: rag-ui)"
	@echo "  HF_TOKEN                 - Hugging Face Token (will prompt if not provided)"
	@echo "  {SAFETY,LLM}             - Model id as defined in values (eg. llama-3-2-1b-instruct)"
	@echo "  {SAFETY,LLM}_URL         - Model URL"
	@echo "  {SAFETY,LLM}_API_TOKEN   - Model API token for remote models"
	@echo "  {SAFETY,LLM}_TOLERATION  - Model pod toleration"

# Create namespace and deploy
namespace:
	oc create namespace $(NAMESPACE) && oc label namespace $(NAMESPACE) modelmesh-enabled=false ||:
	oc project $(NAMESPACE) ||:

.PHONY: install-mcp-servers
install-mcp-servers: namespace

	@echo "Deploying Helm chart $(MCP_SERVERS_CHART_PATH) as release $(MCP_SERVERS_RELEASE_NAME) in namespace $(NAMESPACE)..."
	helm upgrade --install $(MCP_SERVERS_RELEASE_NAME) $(MCP_SERVERS_CHART_PATH) -n $(NAMESPACE)

.PHONY: configure-pipeline-server
configure-pipeline-server: namespace

	@echo "Deploying Helm chart $(CONFIGURE_PIPELINE_SERVER_CHART_PATH) as release $(CONFIGURE_PIPELINE_SERVER_RELEASE_NAME) in namespace $(NAMESPACE)..."
	helm upgrade --install $(CONFIGURE_PIPELINE_SERVER_RELEASE_NAME) $(CONFIGURE_PIPELINE_SERVER_CHART_PATH) -n $(NAMESPACE)


set_ingestion_args = \
	$(if $(SOURCE),--set source='$(SOURCE)',) \
	$(if $(EMBEDDING_MODEL),--set embedding_model='$(EMBEDDING_MODEL)',) \
	$(if $(INGESTION_PIPELINE_NAME),--set name='$(INGESTION_PIPELINE_NAME)',) \
	$(if $(INGESTION_PIPELINE_VERSION),--set version='$(INGESTION_PIPELINE_VERSION)',) \
    $(if $(SOURCE),--set-json S3='$(call S3_TEMPLATE,$(ACCESS_KEY_ID),$(SECRET_ACCESS_KEY),$(BUCKET_NAME),$(ENDPOINT_URL),$(REGION))',)

.PHONY: create-ingestion-pipeline
create-ingestion-pipeline: namespace

	@$(eval INGESTION_ARGS := $(call set_ingestion_args))

	@echo "Deploying Helm chart $(INGESTION_PIPELINE_CHART_PATH) as release $(INGESTION_PIPELINE_RELEASE_NAME) in namespace $(NAMESPACE)..."
	helm upgrade --install $(INGESTION_PIPELINE_RELEASE_NAME) $(INGESTION_PIPELINE_CHART_PATH) -n $(NAMESPACE) $(INGESTION_ARGS) ||:

list-models-%:
	@helm template dummy-release $(LLM_SERVICE_CHART_PATH) --set _debugListModels=true --values $(LLM_SERVICE_CHART_PATH)/values-$*.yaml |grep ^model

.PHONY: list-models
list-models: list-models-gpu

helm_llm_service_args = \
    $(if $(LLM),--set-json models.$(LLM).enabled='true',) \
    $(if $(SAFETY),--set-json models.$(SAFETY).enabled='true',) \
    $(if $(LLM_TOLERATION),--set-json models.$(LLM).inferenceService.tolerations='$(call TOLERATIONS_TEMPLATE,$(LLM_TOLERATION))',) \
    $(if $(SAFETY_TOLERATION),--set-json models.$(SAFETY).inferenceService.tolerations='$(call TOLERATIONS_TEMPLATE,$(SAFETY_TOLERATION))',)

install-llm-service-%: namespace
	@$(eval HELM_ARGS := $(call helm_llm_service_args))

	@echo "Deploying Helm chart $(LLM_SERVICE_CHART_PATH) as release $(LLM_SERVICE_RELEASE_NAME) in namespace $(NAMESPACE)..."; \
	helm upgrade --install $(LLM_SERVICE_RELEASE_NAME) $(LLM_SERVICE_CHART_PATH) -n $(NAMESPACE) --values $(LLM_SERVICE_CHART_PATH)/values-$*.yaml \
		--set secret.hf_token=$(HF_TOKEN) $(HELM_ARGS) $(EXTRA_HELM_ARGS)
	@echo "Waiting for model services to deploy. It will take around 10-15 minutes depending on the size of the model..."
	oc wait -n $(NAMESPACE) --for=condition=Ready --timeout=60m inferenceservice --all ||:

.PHONY: install-llm-service
install-llm-service: install-llm-service-gpu

helm_llama_stack_args = \
    --set pgvector.secret.user=$(POSTGRES_USER) \
    --set pgvector.secret.password=$(POSTGRES_PASSWORD) \
    --set pgvector.secret.dbname=$(POSTGRES_DBNAME) \
    --set minio.secret.user=$(MINIO_USER) \
    --set minio.secret.password=$(MINIO_PASSWORD) \
    $(if $(LLM),--set-json llama-stack.models.$(LLM).enabled='true',) \
    $(if $(SAFETY),--set-json llama-stack.models.$(SAFETY).enabled='true',) \
    $(if $(LLM_URL),--set-json llama-stack.models.$(LLM).url='"$(LLM_URL)"',) \
    $(if $(SAFETY_URL),--set-json llama-stack.models.$(SAFETY).url='"$(SAFETY_URL)"',) \
    $(if $(LLM_API_TOKEN),--set-json llama-stack.models.$(LLM).apiToken='"$(LLM_API_TOKEN)"',) \
    $(if $(SAFETY_API_TOKEN),--set-json llama-stack.models.$(SAFETY).apiToken='"$(SAFETY_API_TOKEN)"',) \
    $(if $(LLAMA_STACK_ENV),--set-json llama-stack.secrets='$(LLAMA_STACK_ENV)',)

.PHONY: create-minio-bucket
create-minio-bucket:
	oc wait -n $(NAMESPACE) --for=condition=Ready pod/minio-0 --timeout=300s
	sleep 5
	oc exec -n $(NAMESPACE) minio-0 -- bash -c "mc alias set local http://localhost:9000 $(MINIO_USER) $(MINIO_PASSWORD) && mc mb local/$(BUCKET_NAME)" ||:

.PHONY: upload-pdf
upload-pdf:
	@echo "Uploading PDF file $(PDF_FILE) to Minio bucket $(BUCKET_NAME)..."
	@test -f "$(PDF_FILE)" || { echo "Error: File $(PDF_FILE) not found"; exit 1; }
	@echo "Waiting for minio-0 pod to be ready..."
	oc wait -n $(NAMESPACE) --for=condition=Ready pod/minio-0 --timeout=60s
	@echo "Creating temporary file in minio pod..."
	cat "$(PDF_FILE)" | oc exec -i -n $(NAMESPACE) minio-0 -- bash -c "cat > /tmp/$(shell basename $(PDF_FILE))"
	@echo "Uploading file to bucket..."
	oc exec -n $(NAMESPACE) minio-0 -- bash -c "mc alias set local http://localhost:9000 $(MINIO_USER) $(MINIO_PASSWORD) && mc cp /tmp/$(shell basename $(PDF_FILE)) local/$(BUCKET_NAME)/ && rm /tmp/$(shell basename $(PDF_FILE))"
	@echo "PDF file $(PDF_FILE) uploaded successfully to Minio bucket $(BUCKET_NAME)"

.PHONY: install-otel-collector
install-rag: namespace install-mcp-servers
	@echo "Deploying OTEL COLLECTOR Helm chart $(OTEL_COLLECTOR_CHART_PATH) as release $(OTEL_COLLECTOR_RELEASE_NAME) in namespace $(NAMESPACE)..."
	helm upgrade --install $(OTEL_COLLECTOR_RELEASE_NAME) $(OTEL_COLLECTOR_CHART_PATH) -n $(NAMESPACE) $(HELM_ARGS) $(EXTRA_HELM_ARGS)

	@$(MAKE) create-minio-bucket
	@$(MAKE) upload-pdf
	@$(MAKE) status

	@echo "Waiting for deployment to be ready..."
	@$(MAKE) wait

.PHONY: install-rag
install-rag: namespace install-mcp-servers
	@$(eval HELM_ARGS := $(call helm_llama_stack_args))

	@echo "Deploying Helm chart $(CHART_PATH) as release $(RELEASE_NAME) in namespace $(NAMESPACE)..."
	helm upgrade --install $(RELEASE_NAME) $(CHART_PATH) -n $(NAMESPACE) $(HELM_ARGS) $(EXTRA_HELM_ARGS)
	  --set secret.dt_api_token=$(DT_API_TOKEN) $(HELM_ARGS) $(EXTRA_HELM_ARGS)

	@$(MAKE) create-minio-bucket
	@$(MAKE) upload-pdf
	@$(MAKE) status

	@echo "Waiting for deployment to be ready..."
	@$(MAKE) wait

install-gpu: install-llm-service-gpu install-rag
install-cpu: install-llm-service-cpu install-rag

.PHONY: install
install: install-gpu

# Uninstall the deployment and clean up
.PHONY: uninstall
uninstall: uninstall-helm-release uninstall-mcp-servers uninstall-ingestion-pipeline uninstall-llm-service remove-secrets remove-pvcs

	@echo "Deleting remaining pods in namespace $(NAMESPACE)"
	oc delete pods -n $(NAMESPACE) --all
	@echo "Checking for any remaining resources in namespace $(NAMESPACE)..."
	@echo "If you want to completely remove the namespace, run: oc delete project $(NAMESPACE)"

	@echo "Remaining resources in namespace $(NAMESPACE):"
	@$(MAKE) status

.PHONY: uninstall-llm-service
uninstall-llm-service:

	@echo "Uninstalling Helm release $(LLM_SERVICE_RELEASE_NAME) from namespace $(NAMESPACE)..."
	helm uninstall $(LLM_SERVICE_RELEASE_NAME) -n $(NAMESPACE) || echo "LLM services are not installed or already removed."

.PHONY: uninstall-mcp-servers
uninstall-mcp-servers:

	@echo "Uninstalling Helm release $(MCP_SERVERS_RELEASE_NAME) from namespace $(NAMESPACE)..."
	helm uninstall $(MCP_SERVERS_RELEASE_NAME) -n $(NAMESPACE) || echo "MCP servers are not installed or already removed."

.PHONY: uninstall-ingestion-pipeline
uninstall-ingestion-pipeline:

	@echo "Uninstalling Helm release $(INGESTION_PIPELINE_RELEASE_NAME) from namespace $(NAMESPACE)..."
	helm uninstall $(INGESTION_PIPELINE_RELEASE_NAME) -n $(NAMESPACE) || echo "Pipelines are not installed or already removed."

	@echo "Uninstalling Helm release $(CONFIGURE_PIPELINE_SERVER_RELEASE_NAME) from namespace $(NAMESPACE)..."
	helm uninstall $(CONFIGURE_PIPELINE_SERVER_RELEASE_NAME) -n $(NAMESPACE) || echo "Pipeline is not configured or already removed."


.PHONY: uninstall-helm-release
uninstall-helm-release:

	@echo "Uninstalling Helm release $(RELEASE_NAME) from namespace $(NAMESPACE)..."
	helm uninstall $(RELEASE_NAME) -n $(NAMESPACE) || echo "Helm release not found or already removed."

.PHONY: remove-secrets
remove-secrets:

	@echo "Removing pipeline secret"
	oc delete secret -n $(NAMESPACE) rag-pipeline-secrets || echo "Secret not found or already removed."

.PHONY: remove-pvcs
remove-pvcs:

	@echo "Removing pgvector and minio PVCs"
	oc get pvc -n $(NAMESPACE) -o custom-columns=NAME:.metadata.name | grep -E '^(pg|minio)-data' | xargs oc delete pvc -n $(NAMESPACE) ||:

# Check deployment status
.PHONY: status
status:
	@echo "Listing pods..."
	oc get pods -n $(NAMESPACE) || true

	@echo "Listing services..."
	oc get svc -n $(NAMESPACE) || true

	@echo "Listing routes..."
	oc get routes -n $(NAMESPACE) || true

	@echo "Listing secrets..."
	oc get secrets -n $(NAMESPACE) | grep huggingface-secret || true

	@echo "Listing pvcs..."
	oc get pvc -n $(NAMESPACE) || true

# Wait for all pods to be ready
.PHONY: wait
wait:

	@echo "Delete failed jobs in namespace $(NAMESPACE)..."
	oc get pods -n $(NAMESPACE) --field-selector=status.phase=Failed -o jsonpath='{range .items[?(@.metadata.ownerReferences[0].kind=="Job")]}{.metadata.namespace}{";"}{.metadata.name}{"\n"}{end}' | while IFS=";" read ns pod; do \
	  echo "Deleting FAILED pod $$pod from namespace $$ns"; \
	  oc delete pod "$$pod" -n "$$ns"; \
	done

	@echo "Waiting for all pods to be ready in namespace $(NAMESPACE)..."
	@end=$$(($$(date +%s)+60)); \
	while [ $$(date +%s) -lt $$end ]; do \
	  not_ready=$$(kubectl get pods --no-headers | grep -vE 'Running|Succeeded|Completed'); \
	  if [ -z "$$not_ready" ]; then \
	    echo "All pods are Ready or Completed."; \
		break; \
	  fi; \
	  sleep 2; \
	done; \
	echo "Timeout: Some pods are not Ready or Completed."; \

	@echo "Verifying routes are accessible..."
	@for route in $(oc get routes -n $(NAMESPACE) -o name); do \
		echo "Checking route ${route}..."; \
		host=$(oc get ${route} -n $(NAMESPACE) -o jsonpath='{.spec.host}'); \
		if [ -n "${host}" ]; then \
			echo "Route hostname: ${host}"; \
			echo "Note: Manual verification of route accessibility is recommended"; \
		else \
			echo "WARNING: No hostname found for ${route}"; \
		fi; \
	done
