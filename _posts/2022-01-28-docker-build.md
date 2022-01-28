2022.01.28

### KivaKit - Docker Build Environment &nbsp;&nbsp; <img src="https://www.state-of-the-art.org/graphics/gears/gears.svg" width="48"/>

[KivaKit](https://www.kivakit.org) 1.2.3 provides a Docker build environment that makes it easy to build KivaKit without 
installing software or configuring the build environment.

#### Launching the Build Environment  &nbsp;&nbsp; <img src="https://www.state-of-the-art.org/graphics/rocket/rocket.svg" width="32"/>


Once Docker has been installed, the [KivaKit Docker build environment](https://github.com/Telenav/kivakit/blob/develop/documentation/building/docker-build-environment.md) can be launched with a few simple commands:

    export KIVAKIT_WORKSPACE=~/workspaces/kivakit
    
    mkdir -p $KIVAKIT_WORKSPACE 
    cd $KIVAKIT_WORKSPACE
    git clone --branch develop https://github.com/Telenav/kivakit.git
    bash $KIVAKIT_WORKSPACE/kivakit/setup/setup-repositories.sh
    
    export KIVAKIT_BUILD_IMAGE=1.2.3-snapshot

    docker run \
        --volume "$KIVAKIT_WORKSPACE:/host/workspace" \
        --volume "$HOME/.m2:/host/.m2" \
        --volume "$HOME/.kivakit:/host/.kivakit" \
        --interactive --tty "jonathanlocke/kivakit:$KIVAKIT_BUILD_IMAGE" \
        /bin/bash

The build environment can build source code in a workspace on the host as well as 
inside Docker, so the first line above specifies the *host* workspace. The next four lines clone 
all the required repositories into that workspace. The second *export* line then specifies 
the [KivaKit Docker tag](https://hub.docker.com/repository/docker/jonathanlocke/kivakit) 
to launch. Finally, the *docker run* command launches the build environment.

### Build Scripts &nbsp;&nbsp; <img src="https://www.state-of-the-art.org/graphics/command-line/command-line.svg" width="32"/>

Several build scripts are available in the Docker build environment, and they are all 
made available on our shell's PATH. To build KivaKit, we would use *kivakit-build.sh*.
Since this script is on our PATH, we can build KivaKit without changing our current working
directory with *cd*:

    kivakit-build.sh

We can see all available KivaKit scripts by typing kivakit- followed by **TAB**. Here are some of the more important scripts:

| KivaKit Script                      | Purpose                                   | 
|-------------------------------------|-------------------------------------------|
| kivakit-version.sh                  | Show KivaKit version                      |
| kivakit-build.sh                    | Build KivaKit                             |
| kivakit-build-documentation.sh      | Build KivaKit documentation               |
| kivakit-git-pull.sh                 | Pull changes from GitHub **               |
| kivakit-git-checkout.sh \[branch]   | Check out the given branch **             |
| kivakit-docker-build-workspace.sh   | Switch between host and Docker workspaces |
| kivakit-feature-start.sh \[branch]  | Start a git flow feature branch **        |
| kivakit-feature-finish.sh \[branch] | Finish a git flow feature branch **       |

** executes the command in each kivakit repository

### Build Workspaces &nbsp;&nbsp; <img src="https://www.state-of-the-art.org/graphics/folder/folder.svg" width="32"/>

When the Docker build environment starts up, it is set up to build a workspace 
provided inside the container. This baseline build should work nearly all of the time, 
because the build environment is standardized, and the build should have worked at 
the time the Docker image was created. Having a common, standardized
build environment provides us with an easy way to determine if there's a problem 
in the source code versus a build environment problem. If the Docker workspace builds
successfully, we can be fairly sure that there is nothing wrong with code, even if 
the host workspace build fails.

The *kivakit-docker-build-workspace.sh* script allows us to switch between the 
Docker and host workspaces. Initially, the workspace inside the Docker container
is set up to build. If we want to switch to the host workspace, we can issue 
this command (use **TAB** to complete the long script name):


    kivakit-docker-build-workspace.sh host

Switching to the host workspace will allow us to do an initial build of the code we
cloned into that workspace above. Once the host workspace has been built, KivaKit projects 
can be imported into an IDE like IntelliJ.

*But how does workspace switching work?*

Notice that the *docker run* command has three Docker *--volume* mounts. 
The *--volume* switch makes a host folder visible inside a Docker container:

    --volume [host-folder]:[docker-folder]
    
So, the *docker run* command we issued above makes these three host folders 
available under the */host* folder inside Docker:

| Docker Mount Folder | Host Folder        |
|---------------------|--------------------|
| /host/workspace     | $KIVAKIT_WORKSPACE |
| /host/.m2           | $HOME/.m2          |
| /host/.kivakit      | $HOME/.kivakit     |

When we start up the Docker build environment, the KIVAKIT_WORKSPACE 
environment variable points to a workspace in the */root* folder inside 
our container, and the symbolic links */root/.m2* and */root/.kivakit* 
point to folders under */root/developer*, also inside our container:

| Docker Reference  | Target Folder            |
|-------------------|--------------------------|
| KIVAKIT_WORKSPACE | /root/workspace          |
| /root/.m2         | /root/developer/.m2      |
| /root/.kivakit    | /root/developer/.kivakit |

When we switch to the host workspace, the KIVAKIT_WORKSPACE environment 
variable and the symbolic links are switched to point to folders that 
have been mapped into the Docker container from the host (using the 
--volume mount switches, as described above). Now our environment
looks like this:

| Docker Reference  | Target Folder   |
|-------------------|-----------------|
| KIVAKIT_WORKSPACE | /host/workspace |
| /root/.m2         | /host/.m2       |
| /root/.kivakit    | /host/.kivakit  |

and, when we execute *kivakit-build.sh* in Docker, the host's workspace
will be built, *using our standardized Docker build environment*.

<br/>

<img src="https://www.state-of-the-art.org/graphics/line/line.svg" width="300"/>

#### Code

The scripts for the KivaKit Docker build environment are here: 

* [Build Scripts](https://github.com/Telenav/kivakit/tree/develop/tools/building)  
* [Setup Scripts](https://github.com/Telenav/kivakit/tree/develop/setup)

<br/>
<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="build-environment"
  theme="github-dark"
  crossorigin="anonymous"
></script>
