---
title: Persistent bash history for ephemeral Docker Compose containers
date: 2022-02-06 16:50:57
tags: [Docker, Docker Compose, Bash]
---

If your local development environment uses Docker Compose, you have probably had to deal with the inconvenience of losing your container _bash_ history. I am going to explain when and why this happens and then suggest solutions that will hopefully suit your development workflow.

Bash history is saved to the text file defined in the `$HISTFILE` environment variable and for most systems defaults to `/home/<username>/.bash_history`. The history file has a line for every command that was executed since last closing a shell. 

_Historically macOS uses bash but since Catalina the default is Z shell. When trying the commands below, make sure you are running a bash shell._

You can check the history file location with `echo $HISTFILE`, then print the last 2 commands of the actual history with `history 2`.
This will show (numbers may vary):

```console
$ history 2
    1  echo $HISTFILE
    2  history
```

If you then try to check the content of the history file,
you see the commands you executed in a previous shell, but the two lines above are missing, or the following error:

```console
cat: can't open '/root/.bash_history': No such file or directory
```

> Remember that _bash_ only saves the commands for a given shell to the history file when it closes. If there was no previous shell or it didn't exit properly, there will simply be no history file.

Knowing this we can start setting up our Docker Compose containers to persist _bash_ history when they are recreated.

I tried using a named volume for storing the history file, but this has 2 limitations:

- Named volumes can't be single files.
- Starting from an empty directory and depending on the container image, permissions can get messed up if you let the container create the history file.

![illustration of a ghostly whale](/images/ghost-whale.jpg)

What I ended up with is a ghost history file that lives on my host system. I suggest you create one per project. I will create a generic one in your home directory just as an example.

You could mount your host history file, but this can cause unwanted truncation because of different `$HISTSIZE` and `$HISTFILESIZE` values between containers and host. Let's assume your host system is configured to infinitely store your history. If you share the history with a container that only keeps the last 20 commands, it will throw away the history of your host system when the container exits.

Create the ghost history file on your host system:

 ```console
 $ touch ~/.docker_bash_history
 ```

After creating the file you can mount it as a single file volume for one or more services:

```yaml
services:
  service:
    image: "bash"
    volumes:
      - ~/.docker_bash_history:/root/.bash_history
    environment:
      # These two variables give you infinite history:
      HISTSIZE: -1
      HISTFILESIZE: -1
```

Let's give it a go by running a one-off command on this bash service.

```console
$ docker-compose run service
$ echo hello world
$ exit
```

This bash image container does not keep running in the background. By using `docker-compose run` you get the same effect as recreating a long running container with `docker-compose up -d` and then executing a command in it with `docker-compose exec service bash`.

Now create a new container of the same service and check the history. The _echo_ and _exit_ commands above should still show up.

```console
$ docker-compose run service
$ history
   1  echo hello world
   2  exit
   3  history
$ exit
```

I don't suggest you use the example Docker Compose config above in the `docker-compose.yml` file of an actual project. It makes assumptions about the presence of the ghost _bash_ history file that will probably not be true for everyone working on the project. Another practical solution would be to add the additional volume and/or environment variables to your local `docker-compose.override.yml` file, which should not be committed to version control.