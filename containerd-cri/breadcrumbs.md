## step: lab on multi-node cluster, node01
## key command: crictl pods / crictl ps -a / crictl inspectp <pod_id>
## key command: nsenter -t <pid> -n ip addr → sees pod's eth0 directly
## concept: pause container PID 2430 holds net:[4026532672] namespace alive
## concept: kube-flannel + calico-node share same namespace → same eth0 192.168.12.134
## concept: eth0@if11721 = veth pair, same mechanism as F1 manual labs
## trap: crictl inspect = container, crictl inspectp = pod (the p matters)
## trap: ctr without --namespace k8s.io won't show K8s pods
## status: T31 subtema 5 complete — retrieval Thursday May 13