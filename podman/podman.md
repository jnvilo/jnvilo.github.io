
[Building, Running, and Managing Containers](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/index)
## build 

```
podman build --tag centos:8 -f Dockerfile
```

## run a command in container
```
podman run --rm registry.redhat.io/ubi8/ubi cat /etc/os-release
```

## run a container 
The container will run the default ENTRYPOINT if not command is given as above.
```
pordman run registry.redhat.io/project/image:latest 
```
We can also pass ports as follows:
```
podman run - 8000:8000 registry.redhat.io/project/image:latest 
```



## exec
```
podman  
```

## listing containers
```
podman ps -a 
```

## inspect containers
```
podman inspect 3482d923dw324 
```

## podman tag
```
podman tag 474ff279782b myrhel8:8.0
```

