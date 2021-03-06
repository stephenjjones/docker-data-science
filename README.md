# Docker for Data Science Guide

## Workflow

- Search for a base docker image to start from on [docker hub](https://hub.docker.com/)
	- [Anaconda on docker hub](https://hub.docker.com/r/continuumio/anaconda/)
	- [Tensorflow on docker hub](tensorflow)
	- [RStudio on docker hub](https://hub.docker.com/r/rocker/rstudio/)
- If you need custom libraries not in the base image, you need to [create a custom DockerFile](#custom-dockerfile) that extends your base docker image
- Build a new docker image from your custom DockerFile `$ docker build --rm -t <username>/<image_name> <dockerfile>`
- Start an instance of your custom docker image, exposing the relevant ports and mounting your local directory
	```
	$ docker run -it -v /Users/stephenjones/Developer/projects/docker-data-science/tf_nbs:/notebooks/jones -p 8889:8888 gcr.io/tensorflow/tensorflow bash
	```
- Code away on your local app directory that was mounted (or on your jupyter notebook exposed on the relevant port, localhost:8899)
	- using the `-it` flags and the `bash` at the end of the `docker run` command above gives you terminal access to the running docker instance.  I use this to check logs or install ad hoc libs that i'm trying out in my project, when i don't want to go through rebuilding a docker image.  When i know i'm keeping a library though, add it to your dockerfile and build a new image
- Version control: commit your your local app directory to github as usual, and include your DockerFile in your git repo


#### Alternative workflow

- In addition to the above, you can commit your docker image to a docker repo (e.g., docker hub). That way others can simply run `$ docker run your-image-name`. This is optional and is usually just done for deployment or for open sourcing your work to docker hub.

## Install Docker

[Install docker for mac](https://store.docker.com/editions/community/docker-ce-desktop-mac?tab=description)

## Finding Docker Images

Most major projects have official docker images on [docker hub](https://hub.docker.com/). If not, popular community maintained ones exist.

- [Anaconda on docker hub](https://hub.docker.com/r/continuumio/anaconda/)
- [Tensorflow on docker hub](tensorflow)
- [RStudio on docker hub](https://hub.docker.com/r/rocker/rstudio/)

## Setup Project

#### Example: Simple
```
$ docker run continuumio/anaconda3
```
- This is all you need to start a docker container
- The first time you run it, docker will check the local image cache, and will see you have not yet downloaded the anaconda3 docker image, it will then fetch it and store it locally for the future


#### Example: Anaconda3 with local volume mounted:
```
$ docker run -it -v /Users/stephenjones/Developer/projects/docker-data-science/app:/opt/app continuumio/anaconda3 bash
```
- `-i` flag: alias for `--interactive` keeps STDIN open even if not attached
- `-t` flag: allocates a pseudo-TTY.
- `-it`: basically when you combine `-i` and `-t` it makes the container start to look like a terminal connection session
- `-v` flag: alias for `--volume` Bind mounts a volume. This basically allows you to connect a local folder on your computer to a folder inside the container.  You can then make changes in either the container or your local and its synced (or rather they are the same files).
	- /absolute/path/to/my/local/folder:/absolute/path/to/folder/in/container
	- You must use absolute paths
- `bash`: the bash at the end opens a bash terminal session in the container	
- [All option flags for docker run](https://docs.docker.com/engine/reference/commandline/run/)
	

#### Example: Anaconda with jupyter notebook running
```
$ docker run -it -v /Users/stephenjones/Developer/projects/docker-ml/src:/opt/docker-ml \
-p 8888:8888 continuumio/anaconda3 /bin/bash -c "/opt/conda/bin/conda install jupyter \
-y --quiet && mkdir /opt/notebooks && /opt/conda/bin/jupyter notebook \
 --notebook-dir=/opt/notebooks --ip='*' --port=8888 --no-browser"
```
- `-p 8889:8888` flag: maps the container's port 8888 to your localhost port 8889
- `-c "a command to run"`: In this example we install jupyter, make a new directory for our notebooks, and tell jupyter to use that directory

#### Example: Run tensorflow in shell
```
$ docker run -it gcr.io/tensorflow/tensorflow bash
```

#### Example: Run tensorflow with jupyter notebooks
```
$ docker run -it -p 8888:8888 gcr.io/tensorflow/tensorflow

# With local folder mounted
$ docker run -it -v /Users/stephenjones/Developer/projects/docker-data-science/tf_nbs:/notebooks/jones -p 8889:8888 gcr.io/tensorflow/tensorflow bash

# Python3 version
docker run -it -p 8889:8888 -v /Users/stephenjones/Developer/projects/docker-data-science/tf_nbs:/notebooks/jones gcr.io/tensorflow/tensorflow:latest-py3
```

- These examples are all CPU versions of tensorflow.  See the docker registery for GPU versions
- [Tensorflow on docker hub](https://hub.docker.com/r/tensorflow/tensorflow/)
- Getting docker to utilize the host GPU can be tricky.  You must use the nvidia-docker image
	- [Tensorflow with Docker and GPU](https://medium.com/@gooshan/for-those-who-had-trouble-in-past-months-of-getting-google-s-tensorflow-to-work-inside-a-docker-9ec7a4df945b)
	- [Nvidia-docker](https://devblogs.nvidia.com/parallelforall/nvidia-docker-gpu-server-application-deployment-made-easy/)

#### Example: RStudio

- [rstudio on docker reference](https://github.com/rocker-org/rocker/wiki/Using-the-RStudio-image)
- You must use the RStudio server web interface instead of the local desktop app. You can access it through the port you map to the container (localhost:8787) in the example below

```
# gets R & RStudio and opens port 9797 for using RStudio server in a web browser, and mounts local volume
docker run -dp 8787:8787 -e ROOT=TRUE -v /Users/jones/Developer/projects/docker-data-science:/home/rstudio/ rocker/rstudio
```
- `-e ROOT=TRUE`: The `-e` flag sets environment variables in the container. The RStudio user does not have access to root by default, so you cannot install binary libs with apt-get.  Setting `ROOT=TRUE` enables root from within RStudio

- There are a few popular R base images
	- rocker/rstudio
	- rocker/hadleyverse
	- rocker/ropensci


## Useful Commands

- The primary way to interact with Docker is through the docker cli
- The [docker docs](https://docs.docker.com/) are pretty good. Best to start there.  I've listed some common commands here for reference.

```
# List running containers
$ docker ps

# List all existing containers (not just running ones)
$ docker ps -a

# Run a docker container (performs docker pull, create, and run in one step)
$ docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

# Stops a container
$ docker stop <container-id>

# Force stops a container
$ docker kill <container-id>

# Starts a container (when you want to start one that has been stopped)
$ docker start my-container

# Remove a container and send sigkill to it first
$ docker rm -f my-container

# Removes an image, not just an instance of a running container
$ docker rmi test-container
```


## Custom DockerFile

**Creating your own containers with custom packages**

A Dockerfile is a short plain text file that is a recipe for a docker image.  If you are using any packages not in a community provided docker image, you should extend (via `FROM`) a community image, and install your packages in your own docker file. Then you can build a container from your own docker file for reproducability. 

The best way to learn how to create one is to just look at examples.  They are really easy to read and understand:

- Examples
	- [r base dockerfile](https://github.com/rocker-org/rocker/blob/master/r-base/Dockerfile)

- Docker file explanations
	- FROM specifies which base image your image is built on
	- CMD which command to run immediately when a container is started from this image, unless you specify a different command when running it
	- ADD copies new files from a source and adds them to the containers filesystem path
	- RUN runs a command inside the container
	- Expose tells docker that the container will listen on the specified port when it starts
	- VOLUME creates a mount point with the specified name and tells docker taht the volume may be mounted by the host

- To build an image from a dockerfile

	```
	$ docker build --rm -t <username>/<image_name> <dockerfile>

	# Send an image to the registry, you must be registered first on docker hub
	docker push <username>/<image_name>
	```
	
	- `-rm` flag: removes intermediate containers
	- `-t` flag: name and optionally a tag in the ‘name:tag’ format

## Docker Compose

Docker compose is used to coordinate multiple containers.  It is used more for deployable systems with multiple interacting containers (i.e., proxy server, web server, db server, etc).

[TODO: Add documentation for docker compose]

## References

[Get started with Docker for Mac](https://docs.docker.com/docker-for-mac/)
