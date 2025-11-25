# AI Voice Assistant Deployment (New OpenShift Cluster)

High-level overview: Deploy a voice assistant that records mic audio in a React frontend, sends it to the Orchestrator, which calls Whisper ASR (speech-to-text), an LLM (OpenAI-style chat endpoint), and Piper TTS (text-to-speech). The Orchestrator returns transcript, answer text, and WAV audio back to the frontend.

This runbook assumes you have the prebuilt images in `images/`:
- `images/whisper-custom-latest.tar`
- `images/piper-custom-latest.tar`
- `images/orchestrator-latest.tar`
- `images/frontend-latest.tar`

Namespace: `ai-voice-assistant`

---

## 0. Cluster verification (info only)
- Login to the new cluster:
  ```bash
  oc login --token=sha256~iERxtHCoMBu08wG1dVK70dRypZ8HH6WS1TnA1XPPa30 --server=https://api.ocp.8nxzw.sandbox447.opentlc.com:6443
  ```
- Verify cluster version/operators:
  ```bash
  oc version
  oc get clusterversion
  oc get co
  ```
- Nodes and GPU inventory:
  ```bash
  oc get nodes -o wide
  oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.nvidia\\.com/gpu}{"\t"}{.status.allocatable.nvidia\\.com/gpu}{"\n"}{end}'
  oc describe node <gpu-node> | grep -A5 "nvidia.com/gpu"
  ```
- LLM/Inference services (if any pre-installed):
  ```bash
  oc get inferenceservices -A
  oc get pods -A | grep -i predictor
  ```

## 1. Prepare project/namespace
```bash
oc new-project ai-voice-assistant   # or oc project ai-voice-assistant
```

## 2. Load images into the internal registry
Obtain registry host (defaultRoute enabled):
```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"defaultRoute":true}}'
REG=default-route-openshift-image-registry.apps.<cluster-domain>
TOKEN=$(oc whoami -t)
podman login --tls-verify=false -u kubeadmin -p "$TOKEN" $REG
```
Mirror local tars to ImageStreams:
```bash
oc create imagestream whisper-custom || true
oc create imagestream piper-custom || true
oc create imagestream orchestrator || true
oc create imagestream frontend || true

oc image mirror --keep-manifest-list --insecure=true \
  "docker-archive:images/whisper-custom-latest.tar" \
  "$REG/ai-voice-assistant/whisper-custom:latest"

oc image mirror --keep-manifest-list --insecure=true \
  "docker-archive:images/piper-custom-latest.tar" \
  "$REG/ai-voice-assistant/piper-custom:latest"

oc image mirror --keep-manifest-list --insecure=true \
  "docker-archive:images/orchestrator-latest.tar" \
  "$REG/ai-voice-assistant/orchestrator:latest"

oc image mirror --keep-manifest-list --insecure=true \
  "docker-archive:images/frontend-latest.tar" \
  "$REG/ai-voice-assistant/frontend:latest"
```

## 3. (Optional) Scale GPU node and label
```bash
oc apply -f 30-gpu-machineset-ocp-tb8c9.yaml   # GPU machineset for this cluster (update name/filters if needed)
oc label node <gpu-node> ai-role=asr --overwrite
```

## 4. Apply storage and helper
```bash
oc apply -f 10-rag-pvc.yaml
oc apply -f 20-rag-helper-pod.yaml
oc cp document.pdf ai-voice-assistant/rag-helper:/data/document.pdf
```

## 5. Deploy Whisper ASR
```bash
oc apply -f 40-whisper-deployment.yaml
oc apply -f 50-whisper-route.yaml
oc delete pod -l app=whisper-asr -n ai-voice-assistant   # refresh to latest image
```
Health:
```bash
oc run whisper-test --image=quay.io/quay/busybox:latest -n ai-voice-assistant --restart=Never --rm -i --command -- sh -c "wget -qO- http://whisper-asr.ai-voice-assistant.svc:80/healthz"
oc run whisper-route-test --image=quay.io/curl/curl:8.9.1 -n ai-voice-assistant --restart=Never --rm -i --command -- sh -c "curl -k https://<whisper-route-host>/healthz"
```

## 6. Deploy Piper TTS
```bash
oc apply -f 60-piper-tts.yaml
oc rollout restart deploy/piper-tts -n ai-voice-assistant
```

## 7. Deploy Orchestrator
Ensure `70-orchestrator.yaml` LLM/Whisper URLs match your cluster:
- Whisper URL: `https://<whisper-route-host>/asr`
- LLM URL (OpenAI chat): `http://llama-32-3b-instruct-predictor.my-first-model.svc.cluster.local:8080/v1/chat/completions` (adjust to your LLM service)
Apply/refresh:
```bash
oc apply -f 70-orchestrator.yaml
oc rollout restart deploy/orchestrator -n ai-voice-assistant
```
Health:
```bash
curl -k https://<orchestrator-route-host>/healthz
```
Smoke `/turn`:
```bash
curl -s -k -X POST https://<orchestrator-route-host>/turn -F "audio=@sample.m4a"
```
Expect: transcript + answer_text + `audio_b64` (WAV).

## 8. Deploy Frontend
Ensure `80-frontend.yaml` env `VITE_ORCHESTRATOR_URL` points to your orchestrator route (`.../turn`).
Apply and restart:
```bash
oc apply -f 80-frontend.yaml
oc rollout restart deploy/frontend -n ai-voice-assistant
```
Route: `https://frontend-<ns>.<cluster-domain>`

## 9. Full health check (order)
```bash
oc get pods -n ai-voice-assistant
curl -k https://<whisper-route-host>/healthz
curl -k https://<orchestrator-route-host>/healthz
curl -k https://<frontend-route-host>   # should return index.html
```

## 10. Smoke test end-to-end
- Use frontend: click Start → Stop & Send → expect transcript/answer and playable audio.
- CLI `/turn` test (as above) to confirm audio_b64 is present.

## 11. Findings/report checklist
- Pods all Running (whisper-asr, piper-tts, orchestrator, frontend, rag-helper).
- Routes responsive (healthz 200).
- `/turn` returns transcript + answer + audio_b64 (WAV).
- Frontend UI loads and plays audio.
- Images mirrored successfully to target registry.

---

## Execution log (new cluster ai-voice-assistant)
- Login: `oc login --token=sha256~iERxtHCoMBu08wG1dVK70dRypZ8HH6WS1TnA1XPPa30 --server=https://api.ocp.8nxzw.sandbox447.opentlc.com:6443`
- Project: `oc new-project ai-voice-assistant`
- Registry external route enabled: `oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"defaultRoute":true}}'`
- Registry route host: `default-route-openshift-image-registry.apps.ocp.8nxzw.sandbox447.opentlc.com`
- Registry auth check: `podman login --tls-verify=false -u kubeadmin -p "$(oc whoami -t)" default-route-openshift-image-registry.apps.ocp.8nxzw.sandbox447.opentlc.com`
- Image mirrors completed (skopeo helper + PVC `skopeo-cache`):
  - `oc apply -f 90-skopeo-cache-pvc.yaml`
  - `oc run skopeo-helper --image=quay.io/skopeo/stable:latest -n ai-voice-assistant --restart=Never --rm -i --tty --overrides='{"spec":{"volumes":[{"name":"cache","persistentVolumeClaim":{"claimName":"skopeo-cache"}}],"containers":[{"name":"skopeo","image":"quay.io/skopeo/stable:latest","command":["sleep","infinity"],"volumeMounts":[{"mountPath":"/cache","name":"cache"}]}]}}'`
  - Inside helper (all succeeded):
    - `skopeo copy docker-archive:/cache/orchestrator-latest.tar docker://image-registry.openshift-image-registry.svc:5000/ai-voice-assistant/orchestrator:latest --dest-tls-verify=false`
    - `skopeo copy docker-archive:/cache/piper-custom-latest.tar docker://image-registry.openshift-image-registry.svc:5000/ai-voice-assistant/piper-custom:latest --dest-tls-verify=false`
    - `skopeo copy docker-archive:/cache/whisper-custom-latest.tar docker://image-registry.openshift-image-registry.svc:5000/ai-voice-assistant/whisper-custom:latest --dest-tls-verify=false`
    - `skopeo copy docker-archive:/cache/frontend-latest.tar docker://image-registry.openshift-image-registry.svc:5000/ai-voice-assistant/frontend:latest --dest-tls-verify=false`
  - Imagestreams created: `oc create imagestream <name>` for whisper-custom, piper-custom, orchestrator, frontend.
- Storage/helper:
  - `oc apply -f 10-rag-pvc.yaml`
  - `oc apply -f 20-rag-helper-pod.yaml`
  - `oc cp document.pdf ai-voice-assistant/rag-helper:/data/document.pdf`
- Deployments applied/refreshed:
  - `oc apply -f 40-whisper-deployment.yaml`
  - `oc apply -f 50-whisper-route.yaml`
  - `oc apply -f 60-piper-tts.yaml`
  - `oc apply -f 70-orchestrator.yaml`
  - `oc apply -f 80-frontend.yaml`
  - `oc rollout restart deploy/piper-tts deploy/orchestrator deploy/frontend -n ai-voice-assistant`
- GPU scale: created machineset `ocp-tb8c9-gpu-us-east-2b` (target 1 GPU worker).
- Nodes ready: `ip-10-0-45-245` (control-plane/worker) and `ip-10-0-57-15` (GPU worker) both Ready.
- GPU/node details:
  - `oc get nodes -o wide` → two Ready nodes (kubelet v1.33.5, RHCOS 9.6.20250925-0).
  - `oc get nodes -o jsonpath='{...gpu}{...kubeletVersion}{...osImage}'` → GPU allocatable 1 on `ip-10-0-45-245` and `ip-10-0-57-15`.
  - `oc describe node ip-10-0-45-245... | grep -A5 -i "Memory"` → NVIDIA L4, gpu.memory=23034, memory ~63.4Gi capacity/~62.2Gi allocatable.
- Scale adjustments: briefly scaled machineset to 2, then deleted extra node (`ip-10-0-32-4`, machine `ocp-tb8c9-gpu-us-east-2b-8jtr2`) and scaled back to 1 GPU worker.
- Commands run for GPU/node setup:
  - `oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}'`
  - Created machineset manifest `30-gpu-machineset-ocp-tb8c9.yaml`
  - `oc apply -f 30-gpu-machineset-ocp-tb8c9.yaml`
  - `oc get machinesets -n openshift-machine-api`
  - `oc get machines -n openshift-machine-api -o wide`
  - `oc get nodes -o wide`
  - `oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\\.com/gpu}{"\n"}{end}'`
  - `oc describe node ip-10-0-45-245.us-east-2.compute.internal | grep -A5 -i "Memory"`
  - Scale/delete cycle: `oc scale machineset ocp-tb8c9-gpu-us-east-2b --replicas=2`, `oc delete machine ocp-tb8c9-gpu-us-east-2b-8jtr2 -n openshift-machine-api`, `oc delete node ip-10-0-32-4.us-east-2.compute.internal`, `oc scale machineset ocp-tb8c9-gpu-us-east-2b --replicas=1`
- Current (post-mirror) verification:
  - `oc project ai-voice-assistant`
  - `oc get pods -o wide` →
    - frontend, orchestrator, piper-tts, rag-helper: Running on ip-10-0-57-15.
    - whisper-asr: (after label/rollout) one pod Running on ip-10-0-57-15; a second replica briefly Pending (check replicas if not needed).
  - `oc get svc,route` → services up; routes: frontend-ai-voice-assistant..., orchestrator-ai-voice-assistant..., whisper-asr-ai-voice-assistant....
  - `oc get nodes -o wide` → ip-10-0-45-245 (control-plane/worker) Ready; ip-10-0-57-15 (worker) Ready.
  - `oc get imagestreams` / `oc get istag` → frontend/orchestrator/piper-custom/whisper-custom latest tags present (sha references recorded).

Pending Whisper fix (to run on bastion):
- Label GPU worker for ASR: `oc label node ip-10-0-57-15.us-east-2.compute.internal ai-role=asr --overwrite` ✅
- Restart whisper: `oc rollout restart deploy/whisper-asr -n ai-voice-assistant` ✅
- Recheck pods: `oc get pods -o wide` (one Running, one Pending; set replicas=1 if needed).
- Health checks to capture (once Pending cleared or scaled to 1):
  - Whisper: `oc run whisper-health --image=quay.io/curl/curl:8.9.1 --restart=Never --rm -i --command -- sh -c "curl -k https://$(oc get route whisper-asr -o jsonpath='{.spec.host}')/healthz"` → `{"status":"ok"}`
  - Orchestrator: `oc run orch-health --image=quay.io/curl/curl:8.9.1 --restart=Never --rm -i --command -- sh -c "curl -k https://$(oc get route orchestrator -o jsonpath='{.spec.host}')/healthz"` → `{"status":"ok"}`
  - Frontend: `oc run fe-health --image=quay.io/curl/curl:8.9.1 --restart=Never --rm -i --command -- sh -c "curl -I -k https://$(oc get route frontend -o jsonpath='{.spec.host}')"` → `HTTP/1.1 200 OK` (nginx)
- Scale whisper to one replica (extra Pending pod removed):
  - `oc scale deploy/whisper-asr --replicas=1 -n ai-voice-assistant`
  - `oc get pods -o wide` → single whisper-asr running on ip-10-0-57-15 alongside frontend/orchestrator/piper-tts/rag-helper (all Running).
- Smoke test: frontend live call succeeded end-to-end (audio round-trip OK via orchestrator/whisper/piper).
- Smoke `/turn` (real clip):
  - Copy `sample.m4a` into a curl pod (`oc cp` or inline) and run:
    `curl -s -k -X POST https://$(oc get route orchestrator -o jsonpath='{.spec.host}')/turn -F "audio=@/tmp/sample.m4a"`
