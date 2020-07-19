
[Building, Running, and Managing Containers](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/index)
## build 

```
podman build --tag centos:8 -f Dockerfile
```

## run a command in container
```
podman run --rm registry.redhat.io/ubi8/ubi cat /etc/os-release
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
