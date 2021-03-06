# Intro

Docker containers enable greater control over project dependencies, increasing portability of software. 

This document is about how to use docker. It does not describe how docker containers work or when to use them.

Clone this repo to run through the example.
I recommend reading this README on the github page because some text have hyperlinks to specific files.

# Table of Contents
1. [Basic](#basics)
2. [Run](#run)
3. [Dockerfile](#dockerfile)
4. [Volumes](#volumes)
5. [Compose](#compose)
6. [Comments](#comments)
7. [Management](#comments)
8. [Misc](#comments)

## Basics

When using docker, you are primarily dealing with images and containers.

- image: a definition of an environment (including all dependencies)
- container: an ephemeral realization of an image in which some software can run. 

Once a container stops running, all information in it is lost. For example, suppose you use an ubuntu image to make a container in which you run bash. Then you execute:
```
apt-get update
apt-get install curl
```
Hooray. Now you have curl in your container.
Now you exit you container. Then you start it up again. Curl is no long there. 

This will make more sense as you go through this tutorial.

Here are some basic commands:

To list images, run:
```
docker images 
```
To list running containers, run:
```
docker ps
```
To list all containers, run:
```
docker ps -a
```

## Run
Try running:
```
docker run -it ubuntu bash
```

This tells docker to start a container using the ubuntu base image, and to run bash in it. The '-it' flags say that you want to interact with bash inside of the container.

You may notice that you are running as root inside of the container. 
Now exit the container using "exit" or CTRL+D.

Suppose you run:
```
docker run ubuntu bash
```

You may notice that nothing appeared to happen. This is because you told docker to start a container with base image ubuntu, run bash, and then call it a day.

In summary, if you do not specify that you want an interactive session (use -it flags), then the container will only do precisely what you tell it to, and then it will exit.

Now might be a good time to play around with the "docker ps", "docker ps -a" and "docker images" commands.

## Dockerfile

In the previous example we used the ubuntu images which was pulled from docker hub. We can also define our own images using a Dockerfile, like [this one here](python_example/Dockerfile)

The comments in this Dockerfile explain a bit more about how a dockerfile works.

To build the image defined by this Dockerfile, go to the root directory of the dockerfile and run:
```
docker build -t tutorial:latest .
```
This tells docker to build the image from the current directory(Hence the '.'), and to tag it with the name "tutorial" and version "latest". You do not need the -t (tag) flag but it helps to keep things organized. 

Running "docker images" you will see your image created by this Dockerfile. Now try running: 
```
docker run tutorial:latest
```

You should see "yay docker".

You can override the CMD instruction at the end of the Dockerfile by including your down instruction via the command line, for example:
```
docker run tutorial:latest echo 'Bad, docker!'
```

*note: We can run echo because the ENTRYPOINT for the image has defaulted to bash. This need not always be the case. Your ENTRYPOINT could also be python if you wanted it to. Read about ENTRYPOINTS [here](https://docs.docker.com/engine/reference/builder/#entrypoint).*

## Volumes

If you want to exchange data between the docker container and your machine or you want to persist data made in the docker container, use volumes. 

Check out [this Dockerfile](volume_example/Dockerfile).

Now navigate to the volume_example folder and build the image defined therein:
```
docker build -t volume_tutorial .
```
Now enter into the container, and run bash interactively:
```
docker run -it volume_tutorial:latest bash
```
Now, while in the container, navigate to the data folder and write a file there:
```
echo 'persist' > please_persist.txt
```
Now exit your container. Re-enter the container using the 
```
docker run -it volume_tutorial:latest bash
```
Again navigate to the /home/data/ folder (you should be placed in the home directory when you enter the container because of the WORKDIR command in the Dockerfile). You will notice that persist_please.txt is not there. That is because containers are ephemeral. 

Now let's mount a volume to our container so that data will persist. 
You should be in the <project-root>/volume_example/ directory on your system. Now run:
```
docker run -it -v $(pwd)/data/:/home/data/ volume_example:latest bash
```
This commands makes it such that the system's <project_root>/volume_example/data directory and the container's /home/data/ directory are synced up. Changes made in one are reflected in the other. Therefore, changes made in this folder of the container persist after the container has stopped running. 

Navigate to the /home/data/ folder in the container, and notice that sometext.txt is there. This is a file synced from the system's <project_root>/volume_example/data folder.

Now write a file there:
```
echo 'persist, please!' > please_persist.txt
```
Now exit the container and navigate to the synced folder on the system (<project_root>/volume_example/data). You will notice that please_persist.txt is there. 

Now, allow the Dockerfile to run its build in command (defined in Dockerfile) without being overridden by bash (passed on command line).
```
docker run volume_example:latest
```
You will notice that the container executes a curl command. However you navigate to the <project_root>/volume_example/data folder, you will not see the html there. 

Run it again, this time mounting the data folder to a volume:
```
docker run -v $(pwd)/data/:/home/data/ volume_example:latest
```
*Note: we do not need the -it flags because we do not require an interactive session in the container.*

Note that volumes need not be synced with a system folder. You can also make named volumes like so:
```
docker run -v volume_example_volume0:/home/data/ volume_example:latest
```
You will not see whale.html with you navigate to your local data folder, but you will see it when you start the container again and navigate to it's /home/data folder. 
```
docker run -it -v volume_example_volume0:/home/data volume_example:latest bash
cd data
```
Then exit the container.

To see a list of volumes in your docker environment, use:
```
docker volume ls
```

## Compose

Once project becomes more complex and require containers which depend upon one another, it becomes necessary to use docker-compose. 

Navigate to the [compose_example folder](compose_example). Take a look at the [docker-compose.yml](compose_example/docker-compose.yml)  file therein. This file provides instructions for running two docker containers based on two images (whose build definitions reside in the flask and nginx folders' respective Dockerfiles).

Viewing the docker-compose.yml file, you will see many arguments for two different services. These two services map to two containers. The two services/containers are linked together on a container network. 

The flask service runs [a simple flask app](compose_example/flask/app.py) and exposes its port 8000 to the container network.  

The nginx service uses [this config file](compose_example/nginx/nginx.conf) to define a reverse proxy which listens to the port 8000 of the flask container and serves it on port 81 of the nginx container. The [docker-compose file](compose_example/docker-compose.yml) then maps nginx's port 81 to port 80 of your system. Then you can see the flask app running on localhost:80.

To build the container network, navigate to <project_root>/compose_example. Then run:
```
docker-compose build
```
This uses the docker-compose.yml file to build the network. Now to get the network running, use:
```
docker-compose up
```
Once all of the containers are up, you should be able to navigate to localhost:80 to see the simple flask app's output. 

To bring the network down, run:
```
docker-compse down
```

I suggest following the given hyperlinks and reading the comments in the files provided (including [docker_compose.yml](compose_example/docker-compose.yml) and [nginx.conf](compose_example/nginx/nginx.conf). Also take a look at the Dockerfiles which define how the images are built for the flask and nginx services/containers.

You can also run the network as a daemon (in the background) by running:
```
docker-compose up -d
```

If you want to see its logs, run:
```
docker-compose logs 
```

## Comments

You may notice that there are a lot of ways to do the same thing when you have the option of using Dockerfiles, the command line "docker run" commands, and docker-compose.

Personally, I tend to define depenencies in Dockerfiles. I run simple one-off conatiners using "docker run". If I know I'm going to be running a container or network of containers multiple times with the same command line parameteres,I use docker-compose. I usually define volumes and networking parameteres(expose and ports) in docker-compose, as opposed to using the Dockerfile. Find what works for you and try to be consistent. Otherwise you can lose track of where you defined what.


## Management
You may find that, as you experiment with docker, you build up a big mess of images and volumes.

Keep your container environment clean by tagging your images and removing containers that you are not longer using. "docker rm" (for removing containers) and "docker rmi" (for removing images) are your friends.   

See a lot of images tagged <none>:<none>? You may have too many dangling images. Remove them with:
```
docker rmi $(docker images --quiet --filter "dangling=true")
```

## Misc

Suppose you want to move all data from one named volume v0 to named voluem v1. Observe:
```
docker run --rm -v v1:/move_to_here/ -v v0:/move_from_here ubuntu cp -r /move_from_here/* /move_to_here/
```
In plain Enlish: 

Docker! Mount my named volumes v0 and v1 to the respective locations /move_from_here/ and /move_to_here/ in a container running the latest  ubuntu image. Then, in the ubuntu container, execute command to recursively copy all contents from /move_from_here/ to /move_to_here/...please. Once you do that, remove the container (the --rm flag).
