## Create a Secret by providing credentials on the command line 
Create this Secret, naming it regcred:

```
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>```

where:

- <your-registry-server> is your Private Docker Registry FQDN. (https://index.docker.io/v1/ for DockerHub)
- <your-name> is your Docker username.
- <your-pword> is your Docker password.
- <your-email> is your Docker email.

## Inspecting the Secret regcred
To understand the contents of the regcred Secret you just created, start by viewing the Secret in YAML format:
```
kubectl get secret regcred --output=yaml
```

The output is similar to this:

```
apiVersion: v1
kind: Secret
metadata:
  ...
  name: regcred
  ...
data:
  .dockerconfigjson: eyJodHRwczovL2luZGV4L ... J0QUl6RTIifX0=
type: kubernetes.io/dockerconfigjson
```

The value of the .dockerconfigjson field is a base64 representation of your Docker credentials.

To understand what is in the .dockerconfigjson field, convert the secret data to a readable format:

```
kubectl get secret regcred --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode
```

## Create a pod that uses the secret

```
pods/private-reg-pod.yaml 

apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: regcred
```

Create a Pod that uses your Secret, and verify that the Pod is running:

```
kubectl apply -f my-private-reg-pod.yaml
kubectl get pod private-reg
```
