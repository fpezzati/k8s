# I am not good at secret
Secrets are kubernetes componentes that ship infos who should be confidential in some way.

Secrets aren't that secret.

`kubectl create secret generic my-secret --from-literal=password=mypasswd` does the job imperatively.

### Opaque secrets
The most common ones. Here is the .yaml you get from previous command:
```
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  password: bXlwYXNzd2Q= # it is "echo -n 'mypasswd' | base64" value
```
This is the not-even-base64 version:
```
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  password: mypasswd
```
You can use that secret in a pod this way:
```
apiVersion: v1
kind: Pod
metadata:
  name: easy-pod
spec:
  containers:
    - name: cont1
      image: busybox
      env:
        - name: password
          valueFrom:
            secretKeyRef:
              name: secret-as-volume
              key: password
  volumes:
    - name: secret-as-volume
      secret:
        secretName: my-secret
```
Other way to consume secret in pod:

Consume secret in deployment:

### Service account token secrets

### Docker config secrets

### Basic authentication secrets

### SSH authentication secrets

### TSL secrets

### Bootstrap token secrets

How to use them in pods and deploys?
