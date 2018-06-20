# Why I need this

A Dockerfile is a list of commands which should be executed in the container. 
What if I want to create a few non trivial dockerfiles which share a significant amount of code. I introduce a YAML configuration file and a Python script which parses the configuration file. 
This is something like https://jsonnet.org/ but for a Dockefile

The original problew was to generate mutliple Dockerfiles with significant overlap. Specifically I needed (cross) build environments to produce kernel modules and STAP modules for different Linux kernels. My approach to the problem was a container wiht Linux header files, Linux kernel symbols, correct tool chain, dependencies. The approach required a tool which generated a Dockerfile on the fly given the kernel version and distribution. 

The main goals:

* Keep multiple dockerfiles in a single place
* All dockerfiles have a consistent structure
* Enforce specific "best" practices like order of operations
* Control number of layers
* Sort the lists of arguments
* Support macros, reduce code duplication
* Switch to a different OS/OS release/different version of a package is trivial thanks to macros
* Generate help, usage tips, run time and source code comments automatically
* Convenient support for generation of shell scripts, README files
* Generate multiple dockerfiles from a single YAML configuration file
* Specify the expected docker environment in the Dockerfile, for example version of the docker, daemon.json
* what else?


# What it does

The Python script converts YAML on the left to the Dockerfile on the right 

```macros:                                                                                             # Generated by https://github.com/larytet/dockerfile-generator on sub-2 172.20.20.180                                                                                                  ```
``` get_release:                                                                                                                                                                                                                                                                              ```
```  - cat /etc/*release                                                                                             # sudo docker build --tag centos7:latest --file /home/arkady/dockerfile-generator/Dockerfile.centos7  .                                                                  ```
```  - gcc --version                                                                                             # sudo docker run --name centos7 --tty --interactive   centos7:latest                                                                                                        ```
```                                                                                             # sudo docker start --interactive centos7                                                                                                                                                     ```
``` build_essential_centos:                                                                                             # Docker configuration:['{\n', '\t"dns": ["172.20.15.1", "8.8.8.8"],\n', '\t"experimental": true,\n', '\t"graph": "/home/arkady/docker"\n', '}\n', '\n']              ```
```  - gcc                                                                                                                                                                                                                                                                                    ```
```  - gcc-c++                                                                                                                                                                                                                                                                                ```
```  - make                                                                                             FROM centos:centos7                                                                                                                                                                   ```
```                                                                                              MAINTAINER automatically generated from /home/arkady/dockerfile-generator/containers.yml                                                                                                     ```
``` make_dir:                                                                                                                                                                                                                                                                                 ```
```  - mkdir -p /etc/docker                                                                                             RUN set +x && `# Generate README file` &&             echo -e 'Generated by https://github.com/larytet/dockerfile-generator on sub-2 172.20.20.180\n\                 ```
```                                                                                                Build the container. See https://docs.docker.com/engine/reference/commandline/build\n\                                                                                                     ```
``` environment_vars:                                                                                               sudo docker build --tag centos7:latest --file $HOME/dockerfile-generator/Dockerfile.centos7  .\n\                                                                         ```
```   - SHARED_FOLDER /etc/docker # I need an env variable referencing a persistent folder                                                                                               Run the previously built container. See https://docs.docker.com/engine/reference/commandline/run\n\  ```
```                                                                                               sudo docker run --name centos7 --network='host' --tty --interactive centos7:latest\n\                                                                                                       ```
```containers:                                                                                               Start the previously run container\n\                                                                                                                                            ```
```                                                                                               sudo docker start --interactive centos7\n\                                                                                                                                                  ```
``` centos7:                                                                                               Connect to a running container\n\                                                                                                                                                  ```
```    base: centos:centos7                                                                                               sudo docker exec --interactive --tty centos7 /bin/bash\n\                                                                                                           ```
```    packager: rpm                                                                                               Save the container for the deployment to another machine. Use 'docker load' to load saved containers\n\                                                                    ```
```    install:                                                                                               sudo docker save centos7 -o centos7.tar\n\                                                                                                                                      ```
```      - $build_essential_centos                                                                                                Remove container to 'run' it again\n\                                                                                                                       ```
```      - rpm-build                                                                                               sudo docker rm centos7\n\                                                                                                                                                  ```
```    run:                                                                                             ' > README                                                                                                                                                                            ```
```      - $get_release                                                                                             ENV SHARED_FOLDER /etc/docker                                                                                                                                             ```
```    env:                                                                                                                                                                                                                                                                                   ```
```      - $environment_vars                                                                                             RUN `# Install packages` && set -x && \                                                                                                                              ```
```                                                                                             	yum -y -v install gcc gcc-c++ make rpm-build && \                                                                                                                                         ```
```                                                                                             	yum clean all && yum -y clean packages                                                                                                                                                    ```
```                                                                                                                                                                                                                                                                                           ```
```                                                                                             RUN `# Execute commands` && set -x  && \                                                                                                                                                      ```
```                                                                                             `# $get_release` && \                                                                                                                                                                         ```
```                                                                                             	cat /etc/*release && \                                                                                                                                                                    ```
```                                                                                             	gcc --version                                                                                                                                                                             ```
```                                                                                                                                                                                                                                                                                           ```

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
