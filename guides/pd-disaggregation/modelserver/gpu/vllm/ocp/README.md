# OpenShift GPU vLLM overlay prerequisites

This OpenShift-specific overlay uses the `llm-d-benchmark` preprocessing script to
configure RDMA environment variables before starting vLLM. **The overlay does not
create the required `llm-d-benchmark-preprocesses` ConfigMap.** Create it in the
same namespace as the model-server pods before applying the overlay.

1. Obtain the `preprocess` directory from the matching
   [llm-d-benchmark](https://github.com/llm-d/llm-d-benchmark) source. It must
   contain `set_llmdbench_environment.py` at its top level. Keep the script
   version compatible with the `ghcr.io/llm-d/llm-d-benchmark:v1.0.0` image
   referenced in `kustomization.yaml`.
2. From a machine with cluster access, set `NAMESPACE` to the namespace where
   you will deploy the model server and `PREPROCESS_DIR` to that directory:

   ```bash
   export NAMESPACE=llm-d
   export PREPROCESS_DIR=/path/to/llm-d-benchmark/preprocess
   test -f "${PREPROCESS_DIR}/set_llmdbench_environment.py"
   kubectl create configmap llm-d-benchmark-preprocesses \
     --namespace "${NAMESPACE}" \
     --from-file="${PREPROCESS_DIR}" \
     --dry-run=client -o yaml | kubectl apply -f -
   ```

3. Verify the ConfigMap exists **before** applying this overlay:

   ```bash
   kubectl get configmap llm-d-benchmark-preprocesses \
     --namespace "${NAMESPACE}" \
     -o go-template='{{index .data "set_llmdbench_environment.py"}}' > /dev/null
   ```

The `preprocess` init container mounts this ConfigMap at `/setup/preprocess`
and executes `/setup/preprocess/set_llmdbench_environment.py`. Without the
ConfigMap, Kubernetes cannot mount the volume and the pod will not initialize.
The overlay also assumes that the SR-IOV network attachments, RDMA resources,
and node-local `/var/model-cache` directory are already provisioned.
