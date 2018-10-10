# What it does

The Python script converts YAML on the left to the Dockerfile on the right 

```
macros:                                                          # Generated by https://github.com/larytet/dockerfile-generator on sub-2 172.20.20.180
 get_release:                                                    
  - cat /etc/*release                                            # sudo docker build --tag centos7:latest --file $HOME/dockerfile-generator/Dockerfile.centos7  .
  - gcc --version                                                # sudo docker run --name centos7 --tty --interactive   centos7:latest
                                                                 # sudo docker start --interactive centos7
 build_essential_centos:                                         # Docker configuration:['{\n', '\t"dns": ["172.20.15.1", "8.8.8.8"],\n', '\t"experimental": true,\n', '\t"graph": "/home/user/docker"\n', '}\n', '\n']
  - gcc                                                          
  - gcc-c++                                                      
  - make                                                         FROM centos:centos7
                                                                 MAINTAINER automatically generated from /home/user/dockerfile-generator/containers.yml
 make_dir:                                                       
  - mkdir -p /etc/docker                                         RUN set +x && `# Generate README file` &&             echo -e 'Generated by https://github.com/larytet/dockerfile-generator on sub-2 172.20.20.180\n\
                                                                   Build the container. See https://docs.docker.com/engine/reference/commandline/build\n\
 environment_vars:                                                 sudo docker build --tag centos7:latest --file $HOME/dockerfile-generator/Dockerfile.centos7  .\n\
   # I need an env variable referencing a persistent folder        Run the previously built container. See https://docs.docker.com/engine/reference/commandline/run\n\
   - SHARED_FOLDER /etc/docker                                     sudo docker run --name centos7 --network='host' --tty --interactive centos7:latest\n\
                                                                   Start the previously run container\n\
containers:                                                        sudo docker start --interactive centos7\n\
                                                                   Connect to a running container\n\
 centos7:                                                          sudo docker exec --interactive --tty centos7 /bin/bash\n\
    base: centos:centos7                                           Save the container for the deployment to another machine. Use 'docker load' to load saved containers\n\
    packager: rpm                                                  sudo docker save centos7 -o centos7.tar\n\
    install:                                                       Remove container to 'run' it again\n\
      - $build_essential_centos                                    sudo docker rm centos7\n\
      - rpm-build                                                ' > README
    run:                                                         ENV SHARED_FOLDER /etc/docker
      - $get_release                                             
    env:                                                         RUN `# Install packages` && set -x && \
      - $environment_vars                                         yum -y -v install gcc gcc-c++ make rpm-build && \
                                                                  yum clean all && yum -y clean packages
                                                                 
                                                                 RUN `# Execute commands` && set -x  && \
                                                                 `# $get_release` && \
                                                                  cat /etc/*release && \
                                                                  gcc --version

```                                                                                                                                                                                                                                                                                              

# Why I need this

A Dockerfile is a list of commands which should be executed in the container. 
What if I want to create a few non trivial dockerfiles which share a significant amount of code. I introduce a YAML configuration file and a Python script which parses the configuration file. 
This is something like https://jsonnet.org/ but for a Dockefile

The original problew was to generate mutliple Dockerfiles with significant overlap. Specifically I needed (cross) build environments to produce kernel modules and STAP modules for different Linux kernels. My approach to the problem was a container wiht Linux header files, Linux kernel symbols, correct tool chain, dependencies. The approach required a tool which generated a Dockerfile on the fly given the kernel version and distribution. 

The main goals:

* Keep multiple dockerfiles in a single place
* All dockerfiles have a consistent structure
* Opens an opportunity to implement template based Dockerfiles generation
* Enforce specific "best" practices like order of operations, running Dockerfile linter
* Simplify control number of layers
* Sort the lists of arguments
* Support macros, reduce code duplication
* Switch to a different OS/OS release/different version of a package is trivial thanks to macros
* Generate help, usage tips, run time and source code comments automatically
* Convenient support for generation of shell scripts, README files
* Generate multiple dockerfiles from a single YAML configuration file
* Specify the expected docker environment in the Dockerfile, for example version of the docker, daemon.json
* Can force order of the Dockerfile updates and reduce the image size
* what else?



# HowTo

Install missing Python packages
```sh
pip install -r requirements.txt
```

Install [DockerCE](https://docs.docker.com/engine/installation/linux/ubuntu/). For example, something like this shall work:

```sh 
# Run as root:

wget https://get.docker.com -O - | sh
systemctl stop docker

# Use overlayFS (because it is cool and recommended)

CONFIGURATION_FILE=$(systemctl show --property=FragmentPath docker | cut -f2 -d=)
cp $CONFIGURATION_FILE /etc/systemd/system/docker.service
perl -pi -e 's/^(ExecStart=.+)$/$1 -s overlay/' /etc/systemd/system/docker.service
systemctl daemon-reload
systemctl start docker
```

Generate dockerfiles (modify file ./containers.yml if necessary)

```sh 
rm -f Dockerfile.* ;./dockerfile-generator.py --config=./containers.yml   
```

Build Docker images. This operation should be done every time ./containers.yml is modified

```sh
for f in ./Dockerfile.*; do filename=`echo $f | sed -E 's/\..Dockerfile.(\S+)/\1/'`;echo Processing $filename;sudo docker build -t $filename -f $f  .;sudo docker save $filename -o $filename.tar;done
```

Check that the artefacts are created
```sh
ls -al Dockerfile* *.tar
```


Stop and remove:
```sh
#docker stop $(docker ps -a -q)
#docker rm $(docker ps -a -q)
#docker rmi $(docker images -q)
./remove-all.sh
```

# Related projects

* https://github.com/avirshup/DockerMake
* http://docteurklein.github.io/2015/01/11/docker-auto-builds-and-me/
* https://blog.dockbit.com/templating-your-dockerfile-like-a-boss-2a84a67d28e9


# TODO

* Support copy_secret which removes the files in the end of the build. The user is expected to "squash" the layer when running the build. 
* The "generate" methods are messy, require refactoring
* Injection of comments is not consistent
* Better management of layers and caching


# Docker in Ubuntu


Add the following line In the /etc/default/docker 
```# Use DOCKER_OPTS to modify the daemon startup options.
DOCKER_OPTS="--dns 172.20.15.1 --dns 8.8.8.8 --dns 8.8.4.4 -g /home/USERNAMEHERE/docker"
```


Create file /etc/docker/daemon.json
```{
	"dns": ["172.20.15.1", "8.8.8.8"],
	"experimental": true,
	"graph": "/home/USERNAME/docker"
}```

