# get the container name in the pod - for target option in next command:
kubectl get pod <problem-pod> -n <namespace-a> -o jsonpath='{.spec.containers[*].name}'

# attach a debug container to the problem container:
kubectl debug -n <namespace-a> -it <problem-pod> --image=nicolaka/netshoot --target=<app-container> --custom=debug-profile.yaml -- sh

# run tcpdump:
tcpdump -i any -nn host <destination-pod-ip> and port <port> -w /tmp/trace.pcap

# get the name of the debug pod:
kubectl get pod <problem-pod> -n <namespace-a> -o jsonpath='{.spec.ephemeralContainers[*].name}'

# copy the pcap from the pod to machine:
kubectl cp -n <namespace-a> <problem-pod>:/tmp/trace.pcap ./trace.pcap -c <debug-pod-name>
