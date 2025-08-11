---
title: "「作って壊して直して学ぶKuternetes」を読んだ"
emoji: ""
type: "tech"
topics: ["kubetnetes", "bbf", "book"]
published: false
---


https://github.com/aoi1/bbf-kubernetes.git

kubectl debug --stdin --tty myapp --image=curlimages/curl:8.4.0 --target=hello-server --namespace default -- sh

kubectl exec -it podname -- command

kubectl --namespace default run curlpod --image=curlimages/curl:8.4.0 --command -- /bin/sh -c "sleep infinity;"

kubectl exec -it curlpod -- sh

kubectl port-forward pod srcport:destport &

## chapter 05

作る ... kubectl apply -f myall.yaml
壊す ... kubectl apply -f pod-desctruction.yaml

kubectl get pod ... status が　CrashLoopBackOffだ、なんでだろう

kubectl describe pod ... reason が　Completed だ、なんでだろう


