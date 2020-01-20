---
title: "Plundering Docker Images"
author: "ropnop"
date: 2016-04-15
summary: "On a recent pentest, we recovered credentials to a private Docker registry. Looting the contained images yielded us source code and admin ssh keys."
toc: true
share_img: "/images/2016/04/B3UHc-5CUAEuSRt.jpg"
tags: ["docker", "pentest"]
---

I'm not a big fan of Docker. Every time I try to use it I get confused, frustrated and annoyed and end up spinning up another VM with vagrant instead. But there's no denying it's becoming more popular and more and more organizations are relying on it.

On a recent pentest, my coworker and I came across credentials to the organization's private Docker registry. I didn't know much (and admittedly still don't) about Docker, but I knew there had to be some juicy files and keys in their Docker images if I could pull them down and sift through them. After looking up Docker commands and scouring SO answers, I ended up getting source code and admin SSH keys from the docker images. I figured I'd share the steps I took for others to reference.

## Docker Registries
We discovered a URL to a private Docker registry and some plaintext credentials. I hadn't used registries before, so I had to look up how they worked and how to access them. I ended up using [this cheatsheet](https://github.com/wsargent/docker-cheat-sheet#registry--repository) which proved invaluable. From the cheatsheet:

> A repository is a hosted collection of tagged images that together create the file system for a container.
> 
> A registry is a host – a server that stores repositories and provides an HTTP API for managing the uploading and downloading of repositories.

The registry we discovered had the URL of:
https://registry.example.com/v2 and required Basic authentication.

To view a list of images in the registry, append "_catalog" to the URL and it will spit out JSON of all the repository and image names:
https://registry.example.com/v2/_catalog

![The Docker registry](/images/2016/04/docker_registry-1.jpg)

Looking the list, we saw some potentially sensitive images, including one called "admin".

### Pulling from a registry
*Quick side note: as much as I love running things from my Mac, Docker is a major pain to run from Mac and requires a VM anyway. Instead, I installed Docker on an Ubuntu VM I had laying around*

To pull down an image from a private registry, you first have to use the Docker 'login' command to authenticate. We had the username/password already, so we entered it and were good to go:
```bash
rflathers@ubuntu:~$ docker login https://registry.example.com/v2/
Username: <username>
Password: <password>
Email: <not necessary>
WARNING: login credentials saved in /home/rflathers/.docker/config.json
Login Succeeded
```
*Note: as Docker nicely warns you, creds are saved in plaintext in config.json. Something to keep in mind during post-exploitation looting*

Once Docker has logged in to the repository, you can then do a `docker pull` to download the image to your host. You have to include the full registry name or Docker will search its public registry for the image. For example, to pull an image named "admin":

```bash
rflathers@ubuntu:~$ docker pull registry.example.com/admin
Using default tag: latest
latest: Pulling from admin
 
abcde9a29090: Pull complete
abcdefafa841: Pull complete
abcde18745c9: Pull complete
abcde3ccebb3: Pull complete
abcde8d47f87: Pull complete
abcde15b336e: Pull complete
abcdefe71cfb: Pull complete
abcdebdf2761: Pull complete
abcdef84677a: Pull complete
abcdefc04499: Pull complete
abcde54eb3fe: Pull complete
abcde656c3a9: Pull complete
abcde78fc125: Pull complete
abcde2baef4c: Pull complete
eabcde3df887: Pull complete
8f4abcde66dc: Pull complete
e7ceabcde9ea: Pull complete
9515abcde66e: Pull complete
Digest: sha256:64dff123453abcde712cef8895465d822308932abcde32a60b776c6cb578751
Status: Downloaded newer image for registry.example.com/admin:latest
```

At this point, the image is downloaded and can be run and containerized. I'm not going to go into running Docker containers here, because I wasn't interested in getting it to run - I just wanted to pillage files from the image.

### From Docker image to Dockerfile
It is possible to derive a Dockerfile from a Docker Image using a publicly available image named, appropriately, [dockerfile-from-image](https://github.com/CenturyLinkLabs/dockerfile-from-image). This is an easy way to see if any custom scripts or commands are added on to the base image.

You have to install this image on the same host and then point it to the image you want the docker file created from. Their readme has you create a useful alias as well:
```bash
$ docker pull centurylink/dockerfile-from-image
$ alias dfimage="docker run -v /var/run/docker.sock:/var/run/docker.sock --rm centurylink/dockerfile-from-image"
$ dfimage --help
Usage: dockerfile-from-image.rb [options] <image_id>
    -f, --full-tree                  Generate Dockerfile for all parent layers
    -h, --help                       Show this message
```
Now to run it, give it the image ID of the one pulled from the private registry. To see a list of imaged IDs, use `docker images`. The ID or full name both work:
```bash
rflathers@ubuntu:~$ dfimage -f 95247d7abcde
```

### Getting Shell Access
To explore the image and discover files, it's usually easiest to just get an interactive shell and explore the filesystem. To do this, you don't actually need to "run" the container for it's main purpose, you can just tell Docker to execute /bin/bash. This will create a container from the image (remember: containers and images are different) and then drop you into a pseudo-TTY.

```bash
rflathers@ubuntu:~$ docker run -i -t --entrypoint /bin/bash 95158d7abcde #image ID
admin@f03f4fabcdef:/$ hostname 
f03f4abcdef #now in shell in container
```

Now you can pillage whatever is in the image. On this particular engagment there were bootstrap scripts (I saw them in the Dockerfile created above) that performed git operations to private repositories. It authenticated over SSH and there was a private key located at "/home/admin/.ssh/id_rsa".

**Re-attaching to exited container.** Once you exit the session, the container will be exited. If you do `docker run` again on the image, a new container will be created and you will "lose" any work or files you created in the old container. To re-attach to the previous container you were in, you have to re-start it and then run `docker exec`.
First you need to find out the container ID you were working in. Running `docker ps -a` will show all exited containers, with the most recent on top. Alternatively, the following command will spit out the ID of the last container created (useful for expansion in commands):
```bash
rflathers@ubuntu:~$ docker ps -l -q
417c6a8abcde
```
Now to get a shell on that container ID again, we restart it and then exec /bin/bash:
```bash
rflathers@ubuntu:~$ docker start 417c6a8abcde && docker exec -i -t 417c6a8abcde /bin/bash
```
Or, to be fancy with expansions and do it in one command:
```bash
rflathers@ubuntu:~$ docker start $(docker ps -l -q) && docker exec -i -t $(docker ps -l -q) /bin/bash
```

### Extracting Files
The easiest ways to get a file from a running container is the [docker cp](https://docs.docker.com/engine/reference/commandline/cp/) command. Since I wanted to just run the command once, as I was exploring the container with shell access I just copied everything I wanted to /tmp/loot and tgz'd it up into one file (`/tmp/loot/myloot.tar.gz`)

When you exit the shell session, the container is exited also, but any files you created are still accessible in the container. To pull files from a container, you nee the container ID, which can be found by running `docker ps -a`. If you just exited, it should be the top one (or just run `docker ps -l -q` to see the last container):

```bash
rflathers@ubuntu:~$ docker ps -a
CONTAINER ID        IMAGE                                    COMMAND                  CREATED             STATUS                         PORTS               NAMES
f03f4fdabcde        95158d7abcde                             "/bin/bash"              4 minutes ago       Exited (0) 4 seconds ago                           backstabbing_bardeen
```

Let's say you have a loot file at /tmp/loot/myloot.tar.gz. Exit the container and note it's container ID. To pull out the file from the container to the host:
```bash
rflathers@ubuntu:~$ docker cp f03f4fdabcde:/tmp/loot/myloot.tar.gz ./myloot.tar.gz
```

This will copy the file from the container to your local host.

If you don't need to work with a specific container and know the location of the file you want, you can create a container and cp the file all in one command:
```bash
rflathers@ubuntu:~$ docker cp $(docker create registry.example.com/admin):/home/admin/.ssh/id_rsa ./admin_ssh_key
```

### Dumping the entire filesystem
It should also be possible to extract the entire filesystem from a Docker container should you not want to explore via shell and copy out individual files. The docker [save](https://docs.docker.com/engine/reference/commandline/save/) and [export](https://docs.docker.com/engine/reference/commandline/export/) commands will dump an image or a container to a tarball (respectively). 

Unfortunately, these tarballs are really only useful to Docker. However, there is a tool called "undocker" (https://github.com/larsks/undocker/) which can extract a useful filesystem from these images. See here for a writeup from the author:
http://blog.oddbit.com/2015/02/13/unpacking-docker-images/

I haven't actually tried this tool yet as I didn't need the full filesystem anyway, but if you've used it or have another recommendation please comment!

-ropnop
