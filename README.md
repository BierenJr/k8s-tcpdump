# Capturing tcpdump from a Kubernetes Pod

This guide walks through attaching an ephemeral debug container to a running pod and capturing network traffic with `tcpdump`, without modifying or restarting the target workload.

## Prerequisites

- `kubectl` configured and authenticated to your cluster
- Sufficient RBAC permissions to use `kubectl debug` and `kubectl cp`
- The cluster must support [ephemeral containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/) (Kubernetes v1.23+)
- A `debug-profile.yaml` if custom security context or resource settings are needed (see [Custom Debug Profiles](#custom-debug-profiles))

---

## Step 1 — Identify the Target Container Name

Before attaching the debug container, retrieve the name of the container you want to share a process namespace with. This name is used in the `--target` flag in the next step.

```bash
kubectl get pod <problem-pod> -n <namespace-a> \
  -o jsonpath='{.spec.containers[*].name}'
```

**Example output:**
```
my-app
```

---

## Step 2 — Attach an Ephemeral Debug Container

Attach a [`nicolaka/netshoot`](https://github.com/nicolaka/netshoot) debug container to the pod. The `--target` flag shares the target container's network and process namespace, which is required for `tcpdump` to see its traffic.

```bash
kubectl debug -n <namespace-a> -it <problem-pod> \
  --image=nicolaka/netshoot \
  --target=<app-container> \
  --custom=debug-profile.yaml \
  -- sh
```

| Flag | Description |
|---|---|
| `--image` | The debug image to use. `nicolaka/netshoot` includes `tcpdump` and other network tools. |
| `--target` | The container whose network namespace to attach to. Use the name from Step 1. |
| `--custom` | Path to a YAML file for custom debug profile settings (resource limits, security context, etc.). |

> **Note:** Once the debug shell opens, proceed immediately to Step 3.

---

## Step 3 — Run tcpdump

Inside the debug container shell, start a packet capture. The example below captures all traffic to/from a specific destination IP and port, and writes it to a `.pcap` file.

```bash
tcpdump -i any -nn host <destination-pod-ip> and port <port> -w /tmp/trace.pcap
```

| Flag | Description |
|---|---|
| `-i any` | Capture on all network interfaces. |
| `-nn` | Disable name/port resolution (faster, avoids DNS noise). |
| `host <destination-pod-ip>` | Filter traffic by destination IP. |
| `port <port>` | Filter by port number. |
| `-w /tmp/trace.pcap` | Write raw packets to a file instead of printing to stdout. |

When you have captured enough traffic, press `Ctrl+C` to stop `tcpdump`, then `exit` the shell.

---

## Step 4 — Get the Ephemeral Container Name

After exiting, retrieve the name of the ephemeral (debug) container that was created. This name is needed for the `kubectl cp` command in the next step.

```bash
kubectl get pod <problem-pod> -n <namespace-a> \
  -o jsonpath='{.spec.ephemeralContainers[*].name}'
```

**Example output:**
```
debugger-xxxx
```

---

## Step 5 — Copy the Capture File to Your Local Machine

Use `kubectl cp` to pull the `.pcap` file from the ephemeral container to your local working directory.

```bash
kubectl cp -n <namespace-a> \
  <problem-pod>:/tmp/trace.pcap ./trace.pcap \
  -c <debug-pod-name>
```

The `-c` flag is required here because the pod now contains more than one container (the original app container plus the ephemeral debug container).

---

## Analyzing the Capture

Open `trace.pcap` locally with [Wireshark](https://www.wireshark.org/) or `tcpdump`:

```bash
# Quick summary in terminal
tcpdump -nn -r ./trace.pcap

# Open in Wireshark
wireshark ./trace.pcap
```

---

## Custom Debug Profiles

The `--custom` flag accepts a YAML file to configure the ephemeral container spec. This is useful for setting resource limits or a security context when the default profile is too restrictive or permissive for your environment.

**Example `debug-profile.yaml`:**
```yaml
env:
  - name: TERM
    value: xterm-256color
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
securityContext:
  capabilities:
    add:
      - NET_ADMIN
      - NET_RAW
```

> `NET_RAW` and `NET_ADMIN` capabilities are typically required for `tcpdump` to capture packets. Ensure your cluster's Pod Security Standards permit these capabilities on the target namespace.

---

## Full Example

```bash
# 1. Get container name
kubectl get pod api-server-7d9f4b -n production \
  -o jsonpath='{.spec.containers[*].name}'
# → api

# 2. Attach debug container
kubectl debug -n production -it api-server-7d9f4b \
  --image=nicolaka/netshoot \
  --target=api \
  --custom=debug-profile.yaml \
  -- sh

# 3. Inside the shell, run tcpdump
tcpdump -i any -nn host 10.0.1.45 and port 8080 -w /tmp/trace.pcap
# ... capture traffic, then Ctrl+C and exit

# 4. Get ephemeral container name
kubectl get pod api-server-7d9f4b -n production \
  -o jsonpath='{.spec.ephemeralContainers[*].name}'
# → debugger-9xk2p

# 5. Copy the pcap locally
kubectl cp -n production \
  api-server-7d9f4b:/tmp/trace.pcap ./trace.pcap \
  -c debugger-9xk2p
```

---

## References

- [Kubernetes Ephemeral Containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)
- [kubectl debug](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_debug/)
- [nicolaka/netshoot](https://github.com/nicolaka/netshoot)
- [tcpdump man page](https://www.tcpdump.org/manpages/tcpdump.1.html)