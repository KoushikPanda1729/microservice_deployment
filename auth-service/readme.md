Step 1: Check Pod Status

  kubectl get pods                                                                                                               
  Result: Pod was in Pending status
                                                                                                                                 
  Step 2: Describe Pod to Find Why

  kubectl describe pod auth-service-5578786c5f-kbfww
  Result: Found error in Events section:
  Warning  FailedScheduling  0/2 nodes are available: 2 Too many pods.

  Step 3: Check Node Pod Limits

  kubectl describe nodes | grep -E "Capacity:|Allocatable:|pods:"
  Result: Each node has max 4 pods limit

  Step 4: List All Pods to See Usage

  kubectl get pods --all-namespaces -o wide
  Result: Found 4 pods on each node (both full):
  - Node 1: aws-node, coredns ×2, kube-proxy = 4
  - Node 2: aws-node, kube-proxy, metrics-server ×2 = 4

  Step 5: Fix - Scale Down Unnecessary Pods

  kubectl scale deployment metrics-server -n kube-system --replicas=1
  Result: Freed 1 pod slot → your auth-service got scheduled