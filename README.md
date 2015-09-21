# Docker Flow

Docker Flow manages workflow running in Docker containers. 

With Docker Flow you can define your workflow in yaml files, docker-flow reads the container requirements and execution steps for each job, and uses docker-compose to define Dockerfiles, container linkings and exposed ports, builds docker images, and runs composed application stack in Docker swarm. 

##### Installation

```bash
git clone https://github.com/dockerflow/docker-flow.git
python setup.py install
```

##### Workflow YAML
A workflow yaml file looks like the following: 

```yaml
name: celery-build-test-flow
description: Workflow for building and testing celery package
volumes:
    - /local/exports/workspace:/workspace
    - /local/exports/builds:/builds
jobs:
    - build-celery-sdist:
        image: python:2.7
        working_dir: /workspace
        steps:
           - set -x
           - cd /workspace
           - rm -rf /workspace/*
           - git clone https://github.com/celery/celery
           - cd celery
           - python setup.py sdist
           - cp /workspace/celery/dist/* /builds
        output: /builds/celery-*.tar.gz
```
- The name of the workflow is required
- Volumes can be defined at workflow level or at job level. Volumes defined at the workflow level are mounted to the main container in every job; volumes at job level are only mounted to the main container for that job. Sharing volumes among jobs is necessary when multiple jobs shares files or artifacts, for example, the job "build" builds a pypi package and copied is to a mounted volume /builds, then the next job "functest" installs the pypi package from /builds and runs functional tests in another container. 
- Multiple jobs can be defined. A job's "image" is the container base image that will be used by docker-compose to build Dockerfile, "steps" are the commands to execute in that container. When docker-flow builds Dockerfile, the steps becomes an entrypoint script that gets install the docker image. See more examples at https://github.com/dockerflow/docker-flow/tree/master/examples/flows .


##### Usage

```bash
docker-flow --help
usage: docker-flow [-h] -f FILE [-b BUILD_DIR]

docker-flow

optional arguments:
  -h, --help            show this help message and exit
  -f FILE, --file FILE  The workflow file
  -b BUILD_DIR, --build-dir BUILD_DIR
                        Directory docker-flow will save Dockerfile, compose
                        files to
```
The workflow file is required, you need to provide path to a workflow yaml files with -f or --file. For example: 
```bash
docker-flow --file ~/github/docker-flow/examples/flows/celery-build-test-tornado.yml
```
The build_dir is ~/.docker-flow/builds by default, and can be set to alternative directories by passing in a directory name with -b or --build-dir. 


##### Dockerfiles and Compose Files
The Dockerfile and compose files generated by running the docker-flow command are save to <build_dir>/<workflow_name>. You can manually run docker-compose using these files. See example below. 


##### Example
```bash
mkvirtualenv dockerflow
git clone https://github.com/dockerflow/docker-flow.git
python setup.py install
docker-flow -f examples/flows/celery-build-test-tornado.yml

Building workflow from examples/flows/celery-build-test-tornado.yml
Running jobs in workflow celery-build-test-flow
Generated entrypoint script /Users/cindy.cao/.docker-flow/build/celery-build-test-tornado/entrypoint-functest-tornado-celery
Created Dockerfile /Users/cindy.cao/.docker-flow/build/celery-build-test-tornado/Dockerfile-functest-tornado-celery
Generated entrypoint script /Users/cindy.cao/.docker-flow/build/celery-build-test-tornado/entrypoint-build-celery-sdist
Created Dockerfile /Users/cindy.cao/.docker-flow/build/celery-build-test-tornado/Dockerfile-build-celery-sdist
Running job functest-tornado-celery with compose /Users/cindy.cao/.docker-flow/build/celery-build-test-tornado/docker-compose-functest-tornado-celery.yml

Building project celerybuildtesttornado

Step 0 : FROM python:2.7
 ---> 7a7d87336a33
Step 1 : ADD entrypoint-functest-tornado-celery /usr/local/bin/entrypoint.sh
 ---> Using cache
 ---> 9f0a8419b4c2
Step 2 : RUN chmod 755 /usr/local/bin/entrypoint.sh
 ---> Using cache
 ---> 90637e583c3c
Step 3 : ENTRYPOINT /usr/local/bin/entrypoint.sh
 ---> Using cache
 ---> 0492e56a2881
Successfully built 0492e56a2881
Running compose project celerybuildtesttornado
Project is_running status is True
Running job build-celery-sdist with compose /Users/cindy.cao/.docker-flow/build/celery-build-test-tornado/docker-compose-build-celery-sdist.yml
Building project celerybuildtesttornado
Step 0 : FROM python:2.7
 ---> 7a7d87336a33
Step 1 : ADD entrypoint-build-celery-sdist /usr/local/bin/entrypoint.sh
 ---> Using cache
 ---> 43ac38a89795
Step 2 : RUN chmod 755 /usr/local/bin/entrypoint.sh
 ---> Using cache
 ---> 3f17208d0e9d
Step 3 : ENTRYPOINT /usr/local/bin/entrypoint.sh
 ---> Using cache
 ---> d59c5d061733
Successfully built d59c5d061733
Running compose project celerybuildtesttornado
Project is_running status is True
Completed workflow celery-build-test-flow
```

Running the workflow Dockerfile-functest-tornado-celery.yml created entrypoint scripts and Dockerfile for 2 jobs: 
- build-celery-sdist 
- functest-tornado-celery

The build directory for the workflow is named after the workflow name, and the entrypoint and Dockerfiles for each job are named after the job: 
```
ls -al ~/.docker-flow/build/celery-build-test-tornado
total 48
drwxr-xr-x  8 cindy.cao  staff  272 Sep 20 22:49 .
drwxr-xr-x  5 cindy.cao  staff  170 Sep 20 22:48 ..
-rw-r--r--  1 cindy.cao  staff  162 Sep 20 23:57 Dockerfile-build-celery-sdist
-rw-r--r--  1 cindy.cao  staff  167 Sep 20 23:57 Dockerfile-functest-tornado-celery
-rw-r--r--  1 cindy.cao  staff  256 Sep 20 23:57 docker-compose-build-celery-sdist.yml
-rw-r--r--  1 cindy.cao  staff  266 Sep 20 23:57 docker-compose-functest-tornado-celery.yml
-rw-r--r--  1 cindy.cao  staff  151 Sep 20 23:57 entrypoint-build-celery-sdist
-rw-r--r--  1 cindy.cao  staff  327 Sep 20 23:57 entrypoint-functest-tornado-celery
```

A generated compose file can be manually re-run:
```
cd ~/.docker-flow/build/celery-build-test-tornado
docker-compose -f docker-compose-build-celery-sdist.yml up
```

