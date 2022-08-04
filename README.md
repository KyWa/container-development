# Container Development - Getting Started

Application development is something that can be done so many different ways its hard to keep track of all the tools that are available. Some just use whatever standard toolset is available while others try to find "3rd party" tools for their chosen language to help compile/build their application. And then there is application development which is going to be built and then run within a Container. The tools for the Container landscape are massive and grows more every day. Some tools are there to make life easy and others are there because we needed a new standard for something (relevant [xkcd](https://xkcd.com/927/)).

I am here to bring some light on a tool that isn't necesssarily meant to be a new standard, but a tool that can help application developers looking to turn their application into something that can be built easily and run in a Container.

## Building a Container Image

There is really only one way to build a Container Image and that is with the use of a `Dockerfile`. This file contains instructions on how to build a Container Image. It typically starts with a `FROM` line which indicates the "base" Container Image to use and then other instructions like `COPY` or `RUN` to copy files into the Container Image or run some commands (such as installing packages) in the Container Image. These instructions that get run during build time are what the resulting Container Image will have "in it". The other important part is what does the resulting Container Image do when it starts? This is typically handled by an `ENTRYPOINT` or `CMD` (or both) instruction in the `Dockerfile` (or inherited from the base image).

Let's look at a basic example of a `Dockerfile` to build a simple application that runs a Python Flask application:

```docker
# We are going to start with the official Python image from Docker Hub
FROM docker.io/python:3.8

# This sets our working directory (where the Container will "start" from)
WORKDIR /python-container

# Getting our applicatoin code into the container
COPY . .

# Install our dependencies
RUN python3 -m pip install -r requirements.txt

# Tell the Container what to do when it starts
CMD [ "python3", "-m" , "flask", "run", "--host=0.0.0.0"]
```

The above `Dockerfile` will result in a Container Image that will start a Flask application:

```sh
user@workstation-$ docker build -t pyflask .
[+] Building 3.3s (9/9) FINISHED
 => [internal] load build definition from Dockerfile                                                               0.0s
 => => transferring dockerfile: 214B                                                                               0.0s
 => [internal] load .dockerignore                                                                                  0.0s
 => => transferring context: 2B                                                                                    0.0s
 => [internal] load metadata for docker.io/library/python:3.8                                                      0.0s
 => [1/4] FROM docker.io/library/python:3.8                                                                        0.1s
 => [internal] load build context                                                                                  0.0s
 => => transferring context: 180.35kB                                                                              0.0s
 => [2/4] WORKDIR /python-container                                                                                0.0s
 => [3/4] COPY . .                                                                                                 0.0s
 => [4/4] RUN python3 -m pip install -r requirements.txt                                                           2.9s
 => exporting to image                                                                                             0.1s
 => => exporting layers                                                                                            0.1s
 => => writing image sha256:7994aab4a965fc6b739456f0c0f738ca3943ccb77f9ff9ad6dbb67aaf82c2514                       0.0s
 => => naming to docker.io/library/pyflask                                                                         0.0s

Use 'docker scan' to run Snyk tests against images to find vulnerabilities and learn how to fix them

user@workstation-$ docker run -d -p 5000 pyflask
8151bd52f0ea875750f7c832de8ab7f37bdd37fef1d657b7b69e6e184070aae9

user@workstation-$ docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                     NAMES
8151bd52f0ea   pyflask   "python3 -m flask ru…"   3 seconds ago   Up 3 seconds   0.0.0.0:62069->5000/tcp   cool_lamarr

user@workstation-$ curl http://localhost:62069
<!doctype html>
<html>
  <head>
    <title>flaskdev</title>
  </head>
  <body>
      <h1>Container Flask</h1>
  </body>
</html

user@workstation-$ docker logs cool_lamarr
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on all addresses (0.0.0.0)
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://127.0.0.1:5000
 * Running on http://172.17.0.2:5000 (Press CTRL+C to quit)
172.17.0.1 - - [27/Jul/2022 12:22:33] "GET / HTTP/1.1" 200 -
```

This is fine and works well for most people who know how to create a `Dockerfile` and put the specific instructions in there. What about those who maybe don't know or are unsure of what they need to do to run their application in a Container?

Enter Source-to-Image, or `s2i`

## What is `s2i`
I am not known for my wordsmithing skills, so I will let the `s2i` repository speak for itself:

<hr>
Source-to-Image (S2I) is a toolkit and workflow for building reproducible container images from source code. S2I produces ready-to-run images by injecting source code into a container image and letting the container prepare that source code for execution. By creating self-assembling builder images, you can version and control your build environments exactly like you use container images to version your runtime environments.
<hr>

The big `tl;dr` of this is an `assemble` script, which has logic to look at a directory (typically cloned from a Git repository) and a `run` script which will handle the running of your application. There are a few ways in which to get to this point and we will cover a most of them.

### Using the `s2i` binary
`s2i` has its own binary that allows you to do a few things. First of which 

#### `s2i build`
This will handle a couple of things, first it will look at the content its given (either by local directory or a git clone) and determine how it should be built. It will then generate a `Dockerfile` which it will then build.

The following will build a Container Image from a local directory
```sh
$ s2i build . registry.access.redhat.com/ubi8/python-38 somefuns2i
```

Same as above, but will pull from a Git repository:
```sh
$ s2i build https://github.com/someuser/somerepo registry.access.redhat.com/ubi8/python-38 somefuns2i
```

#### `s2i generate`
What if you don't want to build the image right away and want to have it handled somewhere else down the line? You can use the `generate` function to generate a `Dockerfile` which can then be handed off to something else (such as `buildah`, `podman` or even `docker` itself).

### Links
* `s2i` - https://github.com/openshift/source-to-image
* `s2i` Deep Dive - https://www.youtube.com/watch?v=flI6zx9wH6M&ab_channel=OpenShift (Somewhat older, but still valid)
* SCLORG - https://www.softwarecollections.org/en/ 
* GitHub SCLORG - https://github.com/sclorg

The following are for reproduceable Docker images in the following languages. Note there are others available, but these are the most common. Check the link above for PHP, Ruby and others.

### Python
https://github.com/sclorg/s2i-python-container

### NodeJS
https://github.com/sclorg/s2i-nodejs-container

### .NET
https://github.com/redhat-developer/s2i-dotnetcore

I am not a .NET dev (or a developer at all really), but I deal with enough teams who do make use of .NET that I would like to at least showcase it here as well.

## Security

Use DIGESTs!
