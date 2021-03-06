# fullmetalupdate-automation-build

This project allows to automate builds for fullmetalupdate. Each time there is a change in a branch the repository https://github.com/FullMetalUpdate/meta-fullmetalupdate-extra, a build will be triggered for all machines which depend of this branch.

This project is using buildbot, running in docker containers. For more documentation about buildbot go to https://docs.buildbot.net/current/tutorial/docker.html.

The particularity of docker in this project is that the buildbot worker container has to run a container from FMU. We are therefore using **DooD (Docker outside of Docker)** by bind-mounting host docker socket. Be aware that containers will be able to start “sibling” containers and not child containers (which can cause volume mouting, device access and network problems). More information on DooD at https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/.

This project has only been tested on Linux, but given that all is running in containers that shouldn't be an issue.

## Prerequisites

You must have install all dependencies of FullMetalUpdate. To see all dependancies have a look at https://www.fullmetalupdate.io/docs/documentation/.

You have also to additionnaly install docker-compose:

```sh
pip install docker-compose
```

Test if docker is running:

```sh
sudo docker run -i busybox /bin/echo Success
```

## Usage

### Start the FMU cloud server

You first have to start the FMU cloud server. This server has to be running because all builds performed with FMU need to push them to the cloud server. 

The start the cloud server please read the "Set up the server" section on the documentation of fullmetalupdate on https://www.fullmetalupdate.io/docs/documentation/.

### Set up buildbot

In a working directory, use git to download the project

```sh
$ git clone https://github.com/lquinet/fmu-automation-build.git
$ cd fmu-automation-build
```

Before launching buildbot you have to configure the variables in the [.env](docker/master/.env) file:

* **BUILDBOT_WORKER_WORKDIR** : working directory of the worker. It will store all build files in that directory 
* **SD_CARD_DEV_PATH** : path of SD card device to flash the image builded. :warning: :warning: BE REALLY CAREFULL WITH THIS PATH, CHOOSING A WRONG DEVICE PATH COULD ENTIRELY DAMAGE YOUR HOST MACHINE (E.G. ROOT PARTITION) :warning: :warning:
* **FMU_CLOUD_HOSTNAME** : hostname of FMU cloud server. If you launched it on you local machine you can get it thanks to the `hostname`command.
* **GITHUB_TOKEN** : GitHub API token to push build status to the repository.

Once you have done it, you can launch buildbot:

```sh
$ cd docker
$ docker-compose up
```
You should now be able to go to http://localhost:8010, where you will see a web page similar to:

![Image not found](images/index.png)

### GitHub integration

A reporter is configured to publishes build status using GitHub Status API. In order to use it, you have to provide a token to your GitHub account via the *GITHUB_TOKEN* variable in the [.env](docker/master/.env) file. To get a token on GitHub go to 
https://github.com/settings/tokens and click on "Generate new token". Be sure to enable enough scopes for your personnal token.

Once you have provided it, you should see build status appearing on your commits like that:

![Image not found](images/GitHub-build-status.png)

### CI/CD badge

The CI badge of all builders are available at http://\<buildbotURL\>/badges/\<buildername\>.svg.

![Image not found](images/CI-badge.png)

## Customisation

### master.cfg

If you want to customise the behavior of buildbot, you have to modify the [master.cfg](master.cfg) file.

In this file you can modify workers behavior, changes sources, schedulers, builders, and so on.

There are some custom variables:

* **REPO_URL**: URL of the repository to track for changes.
* **BRANCH_MACHINE_PAIRS**: Defines the branches of *REPO_URL* to track and the dependant machines.
* **YOCTO_REPO_URL**: URL of the local server repository of FMU.

```python
REPO_URL="https://github.com/lquinet/meta-fullmetalupdate-extra.git"
SUPPORTED_MACHINES_ROCKO=["imx6qdlsabresd", "raspberrypi3"]
SUPPORTED_MACHINES_THUD=["imx8mqevk", "stm32mp1-disco"]
SUPPORTED_MACHINES_WARRIOR=["imx8mqevk"]
BRANCH_MACHINE_PAIRS = {
    "rocko" : SUPPORTED_MACHINES_ROCKO,
    "thud" : SUPPORTED_MACHINES_THUD,
    "warrior" : SUPPORTED_MACHINES_WARRIOR,
}
YOCTO_REPO_URL="https://github.com/lquinet/fullmetalupdate-yocto-demo.git"
```

### docker-compose.yml and .env

You can also change other parameters in this file such as BUILDBOT_WEB_PORT (port of web page), BUILDBOT_WORKER_PORT, etc.

The [docker-compose.yml](docker/master/docker-compose.yml) file depends on the [.env](docker/master/.env) file.

## Improvements

* **Deploy** : Automate SD card image flashing as soon as a new image is builded
* **Track other repositories** than https://github.com/FullMetalUpdate/meta-fullmetalupdate-extra. Typically all repositories defined by the manifests in https://github.com/FullMetalUpdate/manifest. This implies to add more change sources (one for each repo) and schedulers (one for each branch)
* **Setup reporters**: build status on GitHub, a mail notifier, an IRC notifier, ...
* **Test**: test if the build is running properly on the target