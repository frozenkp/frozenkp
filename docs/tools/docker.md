# Docker

## Dockerfile

```dockerfile
FROM ubuntu

MAINTAINER frozenkp

WORKDIR /root

RUN apt-get update
```

### build

```bash
docker build -t "name:tag" .
```

### tag

```Bash
docker tag "name:tag" "name:tag"
```

### push

```bash
docker push name:tag
```

## Standard

### run

```bash
docker run -it {--name pwn_env} {-v /??/data:/root/data} {-p "host:container"} {--privileged} image_name /bin/bash
```

- `--name` : container name
- `-v` : volume
- `-p` : port
- `--privileged` : ptrace privilege

### exec

```bash
docker exec -it container_name /bin/bash
```

### cp

```bash
# from container to host
docker cp container_name:src_path dest_path
# from host to container
docker cp stc_path container_name:src_path
```

### start

```bash
docker start container_name
```

### stop

```
docker stop container_name
```

