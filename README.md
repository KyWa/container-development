# Container Development - Getting Started

Application development is something that can be done so many different ways its hard to keep track of all the tools that are available. Some just use whatever standard toolset is available while others try to find "3rd party" tools for their chosen language to help compile/build their application. And then there is application development which is going to be built and then run within a Container. The tools for the Container landscape are massive and grows more every day. Some tools are there to make life easy and others are there because we needed a new standard for something (relevant [xkcd](https://xkcd.com/927/)).

I am here to bring some light on a tool that isn't necesssarily meant to be a new standard, but a tool that can help application developers looking to turn their application into something that can be built easily and run in a Container.

**NOTE** Please note there are some starting knowledge required to make use of this post. If you are unfamiliar with building Container Images, I'd recommend taking a look at the 4th blog post in the MineOps series here for a better understanding: https://blog.kywa.io/mineops-part-4

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
CMD ["python3", "-m" , "flask", "run", "--host=0.0.0.0"]
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

user@workstation-$ docker logs 8151bd52f0ea
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

The big **tl;dr** of `s2i` is you can build your application with no need to worry about how to actually build it. With `s2i` you get a few components to handle things for you. The first is an `assemble` script which handles the building of the application. The next is a `run` script that handles the running of the application. The `run` script has logic to essentially see what packages exist and then determine which path to start the application (primarily a Python situation). The logic in both scripts is fairly straightforward and works for most situations, but can also be expanded upon by creating your own scripts which allows you to add extra logic into them. We will look at how to customize this towards the end of this post.

### Using the `s2i` binary

Since we talked about not needing to know how to actually build your application, lets do this without a Dockerfile first. The `s2i` binary allows you to do a few things. First of which and the most important to us, is a build. It will make use of `docker` or `podman`, whichever it finds in your `PATH` and then run a build with what is specified. The other common alternative, and this is useful for implementing this into a CI/CD system of some kind, is a `generate` function. This will "do" the `s2i` process, but only up to generating a `Dockerfile` which can then be used or customized further.

#### `s2i build`

This will handle a couple of things, first it will look at the content its given (either by local directory or a git clone) and determine how it should be built. It will then generate a `Dockerfile` which it will then build.

The following will build a Container Image from a local directory
```sh
$ s2i build . registry.access.redhat.com/ubi8/python-38 somefuns2i
---> Installing application source ...
---> Installing dependencies ...
Collecting click==8.1.2
Downloading click-8.1.2-py3-none-any.whl (96 kB)
Collecting Flask==2.1.1
Downloading Flask-2.1.1-py3-none-any.whl (95 kB)
Collecting itsdangerous==2.1.2
Downloading itsdangerous-2.1.2-py3-none-any.whl (15 kB)
Collecting Jinja2==3.1.1
Downloading Jinja2-3.1.1-py3-none-any.whl (132 kB)
Collecting MarkupSafe==2.1.1
Downloading MarkupSafe-2.1.1-cp38-cp38-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (25 kB)
Collecting Werkzeug==2.1.1
Downloading Werkzeug-2.1.1-py3-none-any.whl (224 kB)
Colleing gunicorn-20.1.0-py3-none-any.whl (79 kB)
Collecting importlib-metadata>=3.6.0
Downloading importlib_metadata-4.12.0-py3-none-any.whl (21 kB)
Requirement already satisfied: setuptools>=3.0 in /opt/app-root/lib/python3.8/site-packages (from gunicorn==20.1.0->-r requirements.txt (line 7)) (41.6.0)
Collecting zipp>=0.5
Downloading zipp-3.8.1-py3-none-any.whl (5.6 kB)
Installing collected packages: zipp, MarkupSafe, Werkzeug, Jinja2, itsdangerous, importlib-metadata, click, gunicorn, Flask
Successfully installed Flask-2.1.1 Jinja2-3.1.1 MarkupSafe-2.1.1 Werkzeug-2.1.1 click-8.1.2 gunicorn-20.1.0 importlib-metadata-4.12.0 itsdangerous-2.1.2 zipp-3.8.1
WARNING: You are using pip version 21.2.3; however, version 22.2.2 is available.
You should consider upgrading via the '/opt/app-root/bin/python3.8 -m pip install --upgrade pip' command.
Build completed successfully
```

Same as above, but will pull from a Git repository:

```sh
$ s2i build https://github.com/someuser/somerepo registry.access.redhat.com/ubi8/python-38 somefuns2i
```

If we check our local images, we will see that we have a new image:

```sh
$ docker images
REPOSITORY       TAG          IMAGE ID       CREATED          SIZE
somefuns2i       latest       e1f3884208c1   34 seconds ago   861MB
```

#### `s2i generate`

What if you don't want to build the image right away and want to have it handled somewhere else down the line such as some CI/CD tool? You can use the `generate` function to generate a `Dockerfile` (but you can call it whatever you want) which can then be handed off to something else (such as `buildah`, `podman` or even `docker` itself).

```sh
$ s2i generate registry.access.redhat.com/ubi8/python-38 Dockerfile
$ cat Dockerfile
FROM registry.access.redhat.com/ubi8/python-38
LABEL "io.openshift.s2i.build.image"="registry.access.redhat.com/ubi8/python-38" \
      "io.openshift.s2i.scripts-url"="image:///usr/libexec/s2i"
      
USER root
# Copying in source code
COPY upload/src /tmp/src
# # Change file ownership to the assemble user. Builder image must support chown command.
RUN chown -R 1001:0 /tmp/src
USER 1001
# # Assemble script sourced from builder image based on user input or image metadata.
# # If this file does not exist in the image, the build will fail.
RUN /usr/libexec/s2i/assemble
# # Run script sourced from builder image based on user input or image metadata.
# # If this file does not exist in the image, the build will fail.
CMD /usr/libexec/s2i/run
```

### Using an `s2i` base image

If you wish to customize the `Dockerfile` for your build, there are base images that Red Hat has created that can be used for free based on the Universasl Base Image (UBI). These images can be found from the Red Hat Container Catalaog located here: https://catalog.redhat.com/software/containers/search

Although there are plenty of languages that have pre-built `s2i` images ready for use, we will be focusing on Python. For reference we will be using this [Python 3.8](https://catalog.redhat.com/software/containers/rhel8/python-38/5dde9cb15a13461646f7e6a2) image for our testing. At the bottom of this post you will find [links](#links) which point to the various files that these images use.

Let's create a Dockerfile to make use of this image (found in the Python SCLORG link down below):

```Docker
FROM registry.access.redhat.com/ubi8/python-38:latest

# Add application sources to a directory that the assemble script expects them
# and set permissions so that the container runs without root access
USER 0
ADD . /tmp/src
RUN /usr/bin/fix-permissions /tmp/src
USER 1001

# Install the dependencies
RUN /usr/libexec/s2i/assemble

# Set the default command for the resulting image
CMD /usr/libexec/s2i/run
```

The neat thing about the above is that we don't have to change anything about our code or change anything in our `Dockerfile`. This `Dockerfile` will stay the same althroughout our work with Python and never really need to change.

Here is the output of building with an image like this:

```
user@workstation-$ docker build -t quay.io/kywa/container-development:latest .
#1 [internal] load build definition from Dockerfile
#1 sha256:355e2d734470eaccba3699918560cf74249427fab8fc8285115d7e37b9a2b02d
#1 transferring dockerfile: 452B done
#1 DONE 0.1s

#2 [internal] load .dockerignore
#2 sha256:f4561062f8f809345cf9a5a1d5a6d5423c7a4da4ee4d612192a431a0793e307e
#2 transferring context: 2B done
#2 DONE 0.1s

#3 [internal] load metadata for registry.access.redhat.com/ubi8/python-39:latest
#3 sha256:ce534b0fd64e6c22a3c7a7247a4d676ce56b73f1d40fc33c3418b830c2abbbc9
#3 DONE 0.6s

#5 [internal] load build context
#5 sha256:2a59b27c38b259862c7e3fd8e68c83e6e5cdde8834f720a528ef6dcd51308060
#5 transferring context: 2.47kB done
#5 DONE 0.0s

#4 [1/4] FROM registry.access.redhat.com/ubi8/python-39:latest@sha256:84639af111ab2b3fd7ca325f0912e405c117df31c976446b2a1902948f65a61d
#4 sha256:a46474a8524b7c703c05ae33a46c18a083294a470688705fe751eb7a26b2b9d4
#4 CACHED

#6 [2/4] ADD . /tmp/src
#6 sha256:57bd845075e1b8191c3629d7ad152b942f2c095798c555198a95876d545eb180
#6 DONE 0.0s

#7 [3/4] RUN /usr/bin/fix-permissions /tmp/src
#7 sha256:8189bedac5eec60275eafe95f067ae5eed3d0015100bf6496467d160526f9b0e
#7 DONE 0.5s

#8 [4/4] RUN /usr/libexec/s2i/assemble
#8 sha256:408bdc819e3ee26297474bac423e5a3f0be03ba1399f7b324c96a346a4928790
#8 0.521 ---> Installing application source ...
#8 0.562 ---> Installing dependencies ...
#8 0.877 Collecting click==8.1.2
#8 0.991   Downloading click-8.1.2-py3-none-any.whl (96 kB)
#8 1.048 Collecting Flask==2.1.1
#8 1.063   Downloading Flask-2.1.1-py3-none-any.whl (95 kB)
#8 1.096 Collecting itsdangerous==2.1.2
#8 1.111   Downloading itsdangerous-2.1.2-py3-none-any.whl (15 kB)
#8 1.168 Collecting Jinja2==3.1.1
#8 1.183   Downloading Jinja2-3.1.1-py3-none-any.whl (132 kB)
#8 1.295 Collecting MarkupSafe==2.1.1
#8 1.308   Downloading MarkupSafe-2.1.1-cp39-cp39-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (25 kB)
#8 1.373 Collecting Werkzeug==2.1.1
#8 1.389   Downloading Werkzeug-2.1.1-py3-none-any.whl (224 kB)
#8 1.435 Collecting gunicorn==20.1.0
#8 1.449   Downloading gunicorn-20.1.0-py3-none-any.whl (79 kB)
#8 1.561 Collecting importlib-metadata>=3.6.0
#8 1.572   Downloading importlib_metadata-4.12.0-py3-none-any.whl (21 kB)
#8 1.586 Requirement already satisfied: setuptools>=3.0 in /opt/app-root/lib/python3.9/site-packages (from gunicorn==20.1.0->-r requirements.txt (line 7)) (50.3.2)
#8 1.645 Collecting zipp>=0.5
#8 1.659   Downloading zipp-3.8.1-py3-none-any.whl (5.6 kB)
#8 1.729 Installing collected packages: zipp, MarkupSafe, Werkzeug, Jinja2, itsdangerous, importlib-metadata, click, gunicorn, Flask
#8 2.113 Successfully installed Flask-2.1.1 Jinja2-3.1.1 MarkupSafe-2.1.1 Werkzeug-2.1.1 click-8.1.2 gunicorn-20.1.0 importlib-metadata-4.12.0 itsdangerous-2.1.2 zipp-3.8.1
#8 2.235 WARNING: You are using pip version 21.2.3; however, version 22.2.2 is available.
#8 2.235 You should consider upgrading via the '/opt/app-root/bin/python3.9 -m pip install --upgrade pip' command.
#8 DONE 2.3s

#9 exporting to image
#9 sha256:e8c613e07b0b7ff33893b694f7759a10d42e180f2b4dc349fb57dc6b71dcab00
#9 exporting layers 0.1s done
#9 writing image sha256:a4e6b75c423b51b4708f5ab99bfc7178e5fe4ac0c37035b869b695e344e532d6 done
#9 naming to quay.io/kywa/container-development:latest done
#9 DONE 0.1s

Use 'docker scan' to run Snyk tests against images to find vulnerabilities and learn how to fix them
```

### Customizing `s2i`

One of the other neat components of `s2i` images is that when it goes to build or run, it detects if your repository has its own `.s2i/` directory and will use whatever scripts you have in there. Sometimes your build may need something other than just language libraries/modules. What if your application needs something else to run, such as ODBC drivers for a database connection? This is how we would do this, by customizing `s2i` with our own scripts that can extend upon what already exists.

If you are already using the `s2i` scripts to build and run your images, you do not have to actually change anything and can keep what is already there. Let's see how to do that:

```sh
user@workstation-$ tree
.
├── Dockerfile
├── .s2i
│   └── bin
│       ├── assemble
│       └── run
├── app
│   ├── __init__.py
│   ├── routes.py
│   └── templates
│       ├── base.html
│       └── index.html
├── main.py
├── requirements.txt
└── wsgi.py
```

[[In]] our repository, we have our main application code as well as a `.s2i/bin` directory with 2 files in it, `assemble` and `run`. These 2 files have different uses but are both picked up by the `ENTRYPOINT` scripts of the existing image. These 2 files will be run at build and run time (`assemble` during build and `run` during runtime), but you can also have these files just be specific customizations you want and then have them run the main `s2i` scripts after or before the customizations. For the `assemble` script, it could look something like this:

```sh
user@workstation-$ cat .s2i/bin/assemble
#!/bin/bash

# insert code sause for OS based depends
# this file can be used in place of a Dockerfile
# to patch s2i

echo "I'm a customization before the actual application build"
echo ""

${STI_SCRIPTS_PATH}/assemble
```

The above `s2i/bin/assemble` script will run during build time and will go and get some RPM file and install it, but right after will carry on with the rest of the `s2i` assemble process.

And the same is done for the `run` script:

```
user@workstation-$ cat .s2i/bin/run
#!/bin/bash

# this file can be used in place of a Dockerfile
# to patch s2i

${STI_SCRIPTS_PATH}/run
```

There is no real limit to what you can customize, just the limits of your imagination. You could even not call the existing `run` script and could have it do something very specific:

```
user@workstation-$ cat .s2i/bin/run
#!/bin/bash
export SOMEVAR=prod
python3 app.py
```

The above is a horrible example, but gets the idea across you could do whatever you need/want to do for your application. Just note this is going to be run during a step where you are a "mortal" user. If you are needing to complete tasks at an OS level, you will need to make use of a `Dockerfile` where you can specify the `USER`.

Using these custimzations looks like this:

```
user@workstation-$ s2i build ./ registry.access.redhat.com/ubi8/python-39:latest
I'm a customization before the actual application build

---> Installing application source ...
---> Installing dependencies ...
Collecting click==8.1.2
Downloading click-8.1.2-py3-none-any.whl (96 kB)
Collecting Flask==2.1.1
Downloading Flask-2.1.1-py3-none-any.whl (95 kB)
Collecting itsdangerous==2.1.2
Downloading itsdangerous-2.1.2-py3-none-any.whl (15 kB)
Collecting Jinja2==3.1.1
Downloading Jinja2-3.1.1-py3-none-any.whl (132 kB)
Collecting MarkupSafe==2.1.1
Downloading MarkupSafe-2.1.1-cp39-cp39-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (25 kB)
Collecting Werkzeug==2.1.1
Downloading Werkzeug-2.1.1-py3-none-any.whl (224 kB)
Collecting gunicorn==20.1.0
Downloading gunicorn-20.1.0-py3-none-any.whl (79 kB)
Collecting importlib-metadata>=3.6.0
Downloading importlib_metadata-4.12.0-py3-none-any.whl (21 kB)
Requirement already satisfied: setuptools>=3.0 in /opt/app-root/lib/python3.9/site-packages (from gunicorn==20.1.0->-r requirements.txt (line 7)) (50.3.2)
Collecting zipp>=0.5
Downloading zipp-3.8.1-py3-none-any.whl (5.6 kB)
Installing collected packages: zipp, MarkupSafe, Werkzeug, Jinja2, itsdangerous, importlib-metadata, click, gunicorn, Flask
Successfully installed Flask-2.1.1 Jinja2-3.1.1 MarkupSafe-2.1.1 Werkzeug-2.1.1 click-8.1.2 gunicorn-20.1.0 importlib-metadata-4.12.0 itsdangerous-2.1.2 zipp-3.8.1
WARNING: You are using pip version 21.2.3; however, version 22.2.2 is available.
You should consider upgrading via the '/opt/app-root/bin/python3.9 -m pip install --upgrade pip' command.
Build completed successfully
```

## Conclusion

This has been a brief look at a technology for building and running Container Images that is outside the norm, but can provide some value to those who may not want or need to know "what" exactly is going on when building an image. I would however highly recommend that you do understand what is actually going on so that if the need arises when things get more complex.

## Links
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
