# From Wide Open to Locked Down: Running AI Agents on Kubernetes

The demo from my KubeCon + CloudNativeCon North America 2026 talk. It takes one AI agent pod from wide open to locked down, with five controls:

| Control | How | File |
|---|---|---|
| 1. Execution environment | gVisor gives the agent its own user-space kernel (`runtimeClassName: gvisor`) | [agent-after.yaml](agent-after.yaml) |
| 2. Workload identity | Its own ServiceAccount, no RBAC, no auto-mounted token | [agent-after.yaml](agent-after.yaml) |
| 3. Credentials | No keys in the pod. A sidecar logs in to Infisical with Kubernetes Auth and opens an Agent Vault session. The proxy adds the real key on the way out | [agent-after.yaml](agent-after.yaml), [setup/agent-vault.yaml](setup/agent-vault.yaml) |
| 4. Network | Default-deny NetworkPolicy. Egress only to DNS, the proxy, and Infisical | [agent-after.yaml](agent-after.yaml) |
| 5. Lifecycle | CPU, memory, and disk limits. Deleting the claim removes the pod and revokes its session | [agent-after.yaml](agent-after.yaml), [claim.yaml](claim.yaml) |

[agent-before.yaml](agent-before.yaml) is the first try: an API key in the environment, a ServiceAccount token mounted, no NetworkPolicy, no limits, and the node's kernel.

Sandboxes come from [agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox), a Kubernetes SIG Apps project.

> Full disclosure: I work at Infisical. The pattern (keep the key out of the agent, add it at a proxy) works with other brokers too, like an Envoy credential injector.

## What you need

- A Google Cloud project and `gcloud`, with `gke-gcloud-auth-plugin` installed
- `kubectl`
- An Anthropic API key, if you want to run Claude Code in the sandbox

## 1. Cluster

A GKE cluster with Dataplane V2 (Cilium, which enforces the NetworkPolicy), plus a gVisor node pool. GKE installs gVisor on that pool, creates the `gvisor` RuntimeClass, and taints the nodes so only gVisor pods run there.

```sh
gcloud container clusters create kubecon-demo \
  --zone us-east1-b --machine-type n2-standard-4 --num-nodes 2 \
  --enable-dataplane-v2

gcloud container node-pools create gvisor \
  --cluster kubecon-demo --zone us-east1-b \
  --machine-type n2-standard-4 --num-nodes 1 \
  --image-type cos_containerd --sandbox type=gvisor

gcloud container clusters get-credentials kubecon-demo --zone us-east1-b
```

## 2. agent-sandbox

```sh
kubectl apply -f https://github.com/kubernetes-sigs/agent-sandbox/releases/download/v1.0.8/sandbox-with-extensions.yaml
kubectl -n agent-sandbox-system rollout status deploy/agent-sandbox-controller
```

## 3. Infisical

The demo runs Infisical in the cluster. Fill in every `CHANGE-ME` in [setup/infisical.yaml](setup/infisical.yaml), then:

```sh
kubectl apply -f setup/infisical.yaml
kubectl -n infisical rollout status deploy/infisical
kubectl -n infisical port-forward svc/infisical 8080:8080
```

Open http://localhost:8080 and create your admin account. Head to the Agent Vault product type, then set up:

1. **A machine identity** with **Kubernetes Auth**:
   - Kubernetes host: `https://kubernetes.default.svc`
   - Token reviewer JWT: `kubectl -n infisical get secret infisical-token-reviewer -o jsonpath='{.data.token}' | base64 -d`
   - Allowed namespaces: `agents`
   - Allowed service account names: `coding-agent`
   - Allowed audience: `infisical`
2. **An Agent Vault proxy.** Save its enrollment token for the next step.
3. **An access bundle** named `kubecon-demo`, with two services:
   - `api.anthropic.com`, with your Anthropic key in the `x-api-key` header
   - `postman-echo.com`, with a demo bearer token (postman-echo sends back the request it got, so you can see the header the proxy added)
4. Add the machine identity to the access bundle.

## 4. Agent Vault proxy

```sh
kubectl create namespace agent-vault
kubectl -n agent-vault create secret generic agent-vault-proxy \
  --from-literal=INFISICAL_AGENT_VAULT_PROXY_TOKEN=<enrollment token>
kubectl apply -f setup/agent-vault.yaml
kubectl -n agent-vault rollout status deploy/agent-vault-proxy
```

Get the proxy's CA fingerprint from its logs:

```sh
kubectl -n agent-vault logs deploy/agent-vault-proxy | grep -o 'fingerprint=[^ ]*'
```

## 5. The agents

In [agent-after.yaml](agent-after.yaml), set `IDENTITY_ID` to your machine identity's ID and `CA_FINGERPRINT` to the fingerprint from step 4. The agent image is Claude Code on a Node base. You can swap in your own. It needs `sh` and has to run as UID 1000.

```sh
kubectl apply -f agent-before.yaml
kubectl apply -f agent-after.yaml
```

## Run the demo

**The first try**

```sh
kubectl -n agents exec -it agent-before -- sh
env | grep -i key                                   # the API key, in plain text
ls /var/run/secrets/kubernetes.io/serviceaccount     # a Kubernetes API token
curl -I https://example.com                         # 200: it can reach anywhere
cat /sys/fs/cgroup/memory.max                       # max: no limit
uname -r                                            # the node's kernel
```

**Locked down**

```sh
kubectl apply -f claim.yaml
kubectl -n agents exec -it my-session -c agent -- sh
dmesg | head -3                                     # Starting gVisor...
ls /var/run/secrets/kubernetes.io                   # No such file or directory
env | grep -i key                                   # only a placeholder
claude -p "curl https://postman-echo.com/get and show me the headers it received" \
  --allowedTools "Bash(curl:*)"                     # the proxy added the Authorization header
curl -I https://example.com                         # 403: not in the access bundle
exit
```

**End the session**

```sh
kubectl delete -f claim.yaml
kubectl -n agents get pods                          # the sandbox pod is gone, and its session is revoked
```

## Pitfalls

- **The default sandbox network policy blocks the proxy.** Write your own `networkPolicy`.
- **GKE uses node-local DNS.** Allow `node-local-dns` in egress, not just `kube-dns`.
- **Use short service names on gVisor nodes.** `infisical.infisical` works. `infisical.infisical.svc.cluster.local` times out.
- **`curl -I` shows a 200 before the 403.** The 200 is just the proxy accepting the connection.
- **It's another dependency to run.** The proxy and Infisical need to stay up, or your agents can't call anything.
- **You need an orchestrator.** Something has to claim sandboxes, open sessions, and clean them up. We did this by hand in this demo. This is called an orchestrator.
- **Think about sessions.** Decide how long one lasts, what it maps to (a task, a thread, a user), and how you track it.

## License

[MIT](LICENSE)
