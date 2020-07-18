# Project settings

There are few project settings which are handy...

## Automatically scan images on push

Enable automated vulnerability scan after each "image push" to the project:
`library`:

```bash
PROJECT_ID=$(curl -s -u "aduser05:admin" -X GET "https://harbor.${MY_DOMAIN}/api/projects?name=library" | jq ".[].project_id")
curl -s -u "aduser05:admin" -X PUT "https://harbor.${MY_DOMAIN}/api/projects/${PROJECT_ID}" -H  "Content-Type: application/json" -d \
"{
  \"metadata\": {
    \"auto_scan\": \"true\"
  }
}"
```

You should see the following in the Web interface:

![Vulnerability scanning - Automatically scan images on push](./harbor_automatically_scan_images_on_push.png
"Vulnerability scanning - Automatically scan images on push")

Tag the image:

```bash
docker tag nginx:1.13.12 harbor.${MY_DOMAIN}/library/nginx:1.13.12
```

Push the container image to Harbor project `library`:

```bash
docker push harbor.${MY_DOMAIN}/library/nginx:1.13.12
```

Output:

```text
The push refers to repository [harbor.mylabs.dev/library/nginx]
7ab428981537: Mounted from my_project/nginx
82b81d779f83: Mounted from my_project/nginx
d626a8ad97a1: Mounted from my_project/nginx
1.13.12: digest: sha256:e4f0474a75c510f40b37b6b7dc2516241ffa8bde5a442bde3d372c9519c84d90 size: 948
```

All images in that repositories should be automatically checked for
vulnerabilities.

## Scan existing application

I'm going to replicate the "bookinfo" application used for testing Istio:
[https://istio.io/docs/examples/bookinfo/](https://istio.io/docs/examples/bookinfo/)
When the replication completes all images should be automatically scanned
because I'm going to replicate everything into `library` project which has
"Automatically scan images on push" feature enabled.

Create new Replication Rule and initiate replication:

```bash
COUNTER=0
for DOCKER_HUB_REPOSITORY in istio/examples-bookinfo-details-v1 istio/examples-bookinfo-ratings-v1 istio/examples-bookinfo-productpage-v1 istio/examples-bookinfo-reviews-v{1..3}; do
  COUNTER=$((COUNTER+1))
  echo "Replicating (${COUNTER}): ${DOCKER_HUB_REPOSITORY}"
  curl -X POST -H "Content-Type: application/json" -u "admin:admin" "https://harbor.${MY_DOMAIN}/api/replication/policies" -d \
    "{
      \"name\": \"Replication of ${DOCKER_HUB_REPOSITORY}\",
      \"type\": \"docker-hub\",
      \"url\": \"https://hub.docker.com\",
      \"description\": \"Replication Rule for ${DOCKER_HUB_REPOSITORY}\",
      \"enabled\": true,
      \"src_registry\": {
        \"id\": 1
      },
      \"dest_namespace\": \"library\",
      \"filters\": [{
        \"type\": \"name\",
        \"value\": \"${DOCKER_HUB_REPOSITORY}\"
      },
      {
        \"type\": \"tag\",
        \"value\": \"1.1*\"
      }],
      \"trigger\": {
        \"type\": \"manual\"
      }
    }"
  curl -X POST -H "Content-Type: application/json" -u "admin:admin" "https://harbor.${MY_DOMAIN}/api/replication/executions" -d "{ \"policy_id\": $((COUNTER+1)) }"
done
```

Output:

```text
Replicating (1): istio/examples-bookinfo-details-v1
Replicating (2): istio/examples-bookinfo-ratings-v1
Replicating (3): istio/examples-bookinfo-productpage-v1
Replicating (4): istio/examples-bookinfo-reviews-v1
Replicating (5): istio/examples-bookinfo-reviews-v2
Replicating (6): istio/examples-bookinfo-reviews-v3
```

![DockerHub Replication](./DockerHub_Replication.svg "DockerHub Replication")

After a while all images used by "bookinfo" application should be replicated
into `library` project and all should be automatically scanned.

![Harbor Project Repository list](./harbor_project_repository_list.png
"Harbor Project Repository list")

![Harbor Project library image list](./harbor_project_library_examples-bookinfo-reviews-v3.png
"Harbor Project library image list")

## Prevent vulnerable images from running

Now there are two container images in the `library` repository:

* `nginx:1.13.12` - which has many vulnerabilities
* `kuard:blue` - which has no vulnerabilities

Turn on the "Prevent vulnerable images from running" feature:

```bash
PROJECT_ID=$(curl -s -u "aduser05:admin" -X GET "https://harbor.${MY_DOMAIN}/api/projects?name=library" | jq ".[].project_id")
curl -s -u "aduser05:admin" -X PUT "https://harbor.${MY_DOMAIN}/api/projects/${PROJECT_ID}" -H  "Content-Type: application/json" -d \
"{
  \"metadata\": {
    \"prevent_vul\": \"true\",
    \"severity\": \"low\"
  }
}"
```

![Harbor - Prevent vulnerable images from running](./harbor_prevent_vulnerable_images_from_running.png
"Harbor - Prevent vulnerable images from running")

## Use image hosted by Harbor in k8s deployment

Create `kuard` deployment and expose it:

```bash
kubectl run kuard --image=harbor.${MY_DOMAIN}/library/kuard-amd64:blue --port=8080 --expose=true --labels="app=kuard" -n mytest
```

Output:

```text
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
service/kuard created
deployment.apps/kuard created
```

Create Ingress for kuard service:

```bash
export APP=kuard
envsubst < ../files/app_ingress.yaml | kubectl create -f -
```

Output:

```text
ingress.extensions/kuard created
```

You should be able to access kuard at [https://kuard.mylabs.dev](https://kuard.mylabs.dev)
and see this:

![Kuard screenshot](./kuard_screenshot.png "Kuard screenshot")

Try the same with `nginx:1.13.12` image:

```bash
kubectl run nginx --image=harbor.${MY_DOMAIN}/library/nginx:1.13.12 --port=80 --expose=true --labels="app=nginx" -n mytest
```

Output:

```text
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
service/nginx created
deployment.apps/nginx created
```

If you check the pods you will see they are not running:

```bash
sleep 10
kubectl -n mytest get pods --selector=app=nginx
```

Output:

```text
NAME                    READY   STATUS             RESTARTS   AGE
nginx-d879bd8db-nmzc8   0/1     ImagePullBackOff   0          25s
```

The details of one of the pods looks like:

```bash
POD_NAME=$(kubectl -n mytest get pods --selector=app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl -n mytest describe pod $POD_NAME
```

Output:

```text{49}
Name:               nginx-d879bd8db-nmzc8
Namespace:          mytest
Priority:           0
PriorityClassName:  <none>
Node:               ip-192-168-56-161.eu-central-1.compute.internal/192.168.56.161
Start Time:         Fri, 19 Jul 2019 12:52:21 +0200
Labels:             app=nginx
                    pod-template-hash=d879bd8db
Annotations:        kubernetes.io/psp: eks.privileged
Status:             Pending
IP:                 192.168.61.206
Controlled By:      ReplicaSet/nginx-d879bd8db
Containers:
  nginx:
    Container ID:
    Image:          harbor.mylabs.dev/library/nginx:1.13.12
    Image ID:
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-5lzmk (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  default-token-5lzmk:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-5lzmk
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason     Age                From                                                      Message
  ----     ------     ----               ----                                                      -------
  Normal   Scheduled  47s                default-scheduler                                         Successfully assigned mytest/nginx-d879bd8db-nmzc8 to ip-192-168-56-161.eu-central-1.compute.internal
  Normal   BackOff    16s (x3 over 45s)  kubelet, ip-192-168-56-161.eu-central-1.compute.internal  Back-off pulling image "harbor.mylabs.dev/library/nginx:1.13.12"
  Warning  Failed     16s (x3 over 45s)  kubelet, ip-192-168-56-161.eu-central-1.compute.internal  Error: ImagePullBackOff
  Normal   Pulling    2s (x3 over 46s)   kubelet, ip-192-168-56-161.eu-central-1.compute.internal  pulling image "harbor.mylabs.dev/library/nginx:1.13.12"
  Warning  Failed     2s (x3 over 46s)   kubelet, ip-192-168-56-161.eu-central-1.compute.internal  Failed to pull image "harbor.mylabs.dev/library/nginx:1.13.12": rpc error: code = Unknown desc = Error response from daemon: unknown: The severity of vulnerability of the image: "high" is equal or higher than the threshold in project setting: "high".
  Warning  Failed     2s (x3 over 46s)   kubelet, ip-192-168-56-161.eu-central-1.compute.internal  Error: ErrImagePull
```

You are not able to run docker images with "High" security issues. You can see
the error message: `The severity of vulnerability of the image: "high" is equal
or higher than the threshold in project setting: "high".`

## Project RBAC settings

YouTube video: [https://youtu.be/2ZIu9XTvsC0](https://youtu.be/2ZIu9XTvsC0)

Create new project called `my_rbac_test_project`

```bash
curl -u "admin:admin" -X POST -H "Content-Type: application/json" "https://harbor.${MY_DOMAIN}/api/projects" -d \
"{
  \"project_name\": \"my_rbac_test_project\",
  \"public\": 0
}"
```

Try to push the kuard image as a "Guest" user:

```bash
echo admin | docker login --username aduser03 --password-stdin harbor.${MY_DOMAIN}
docker tag gcr.io/kuar-demo/kuard-amd64:blue harbor.${MY_DOMAIN}/my_rbac_test_project/kuard-amd64:blue
docker push harbor.${MY_DOMAIN}/my_rbac_test_project/kuard-amd64:blue
```

Output:

```text{9}
WARNING! Your password will be stored unencrypted in /home/pruzicka/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
The push refers to repository [harbor.mylabs.dev/my_rbac_test_project/kuard-amd64]
656e9c47289e: Preparing
bcf2f368fe23: Preparing
denied: requested access to the resource is denied
```

* Guests are not allow to push anything into the projects as you can see from
  the error message: `denied: requested access to the resource is denied`.

Add user `aduser03` on the project `my_rbac_test_project` as a Developer:

```bash
PROJECT_ID=$(curl -s -u "admin:admin" -X GET "https://harbor.${MY_DOMAIN}/api/projects?name=my_rbac_test_project" | jq ".[].project_id")
curl -u "admin:admin" -X POST "https://harbor.${MY_DOMAIN}/api/projects/${PROJECT_ID}/members" -H "Content-Type: application/json" -d \
"{
  \"role_id\": 2,
  \"member_user\": {
    \"username\": \"aduser03\"
  }
}"
```

![Harbor - Project add new member](./harbor_project_new_member.png
"Harbor - Project add new member")

![Harbor - Project members](./harbor_project_members.png
"Harbor - Project members")

Push the container image again:

```bash
docker push harbor.${MY_DOMAIN}/my_rbac_test_project/kuard-amd64:blue
```

Output:

```text
The push refers to repository [harbor.mylabs.dev/my_rbac_test_project/kuard-amd64]
656e9c47289e: Mounted from library/kuard-amd64
bcf2f368fe23: Mounted from library/kuard-amd64
blue: digest: sha256:1ecc9fb2c871302fdb57a25e0c076311b7b352b0a9246d442940ca8fb4efe229 size: 739
```

Now the image was successfully uploaded:

![Harbor - Project - my_rbac_test_project](./harbor_my_rbac_test_project.png
"Harbor - Project - my_rbac_test_project")

```bash
cd ..
```
