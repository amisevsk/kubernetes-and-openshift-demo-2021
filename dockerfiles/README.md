# Description

This is a simple docker image that deploys apache and just serves up whatever we put in `/usr/local/apache2/htdocs/`

Initially, we just have a basic HTML landing page so that there's something to show. By the end of the demo, we'll have created a configmap that gets mounted into `/usr/local/apache2/htdocs/my-files` to change what's served from that path.

## Building it yourself

The Dockerfile in this repo can be built using
```
podman build -t <your-image-tag> .
```

By default, the server is configured to listen on port 8080. We need to do this because on OpenShift, containers are forbidden from running as root, and only root users can bind to ports <1000. You'll commonly see containers listen on port 8080 instead of 80, and 8443 instead of 443 (for https connections). This usually isn't a big deal, since we can have routes and ingresses listen on 80/443 and route to services/pods that listen on *any* port.

## Running it locally
Once built, you can test this container by running it locally:
```
podman run --rm -p 8080:8080 <your-image-tag>
```
The command above specifies
- `--rm` to remove the container once it terminates (so that it doesn't just stick around)
- `-p 8080:8080` to bind port `8080` in the container to port `8080` on your local machine.

Once running, you can access the (simple) demo application at `http://localhost:8080`

If you'd like to look at the filesystem inside the container, you can use the command
```
podman run -it --rm <your-image-tag> -- /bin/bash
```
Here, we're specifying arguments
- `-i` (or `--interactive`) to connect our terminal to the pod
- `-t` (or `--tty`) to create a pseudo-tty for the container
- `--rm` as before
- a plain `--` to separate arguments from commands
- `/bin/bash` to run `bash` inside the container