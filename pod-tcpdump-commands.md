# K8s Pod — tcpdump Quick Reference

## 1. Get the target container name
```bash
kubectl get pod <problem-pod> -n <namespace> \
  -o jsonpath='{.spec.containers[*].name}'
```

## 2. Attach a debug container
```bash
kubectl debug -n <namespace> -it <problem-pod> \
  --image=nicolaka/netshoot \
  --target=<app-container> \
  --custom=debug-profile.yaml \
  -- sh
```

## 3. Capture traffic
```bash
tcpdump -i any -nn host <destination-pod-ip> and port <port> -w /tmp/trace.pcap
```
Press `Ctrl+C` to stop, then `exit` the shell.

## 4. Get the ephemeral container name
```bash
kubectl get pod <problem-pod> -n <namespace> \
  -o jsonpath='{.spec.ephemeralContainers[*].name}'
```

## 5. Copy the capture file locally
```bash
kubectl cp -n <namespace> <problem-pod>:/tmp/trace.pcap ./trace.pcap \
  -c <debug-container-name>
```