# args vs commands in containers
Let's say we have this one:
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox
  name: busybox
spec:
  containers:
  - args:
    - /bin/sh
    - -c
    - echo hello;sleep 3600
    image: busybox
    name: busybox
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Why use `args` in place of `command`? `command` overrides what docker image provides as `entrypoint`, so, no more docker`entrypoint` or `cmd`. Using `args` will
