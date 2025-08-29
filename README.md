# Install FAS on a Rocky 8 container

## Purpose
1. [Run a container on WSL with the Rocky 8 image](#step-1-run-a-container-on-wsl-with-the-rocky-8-image)
2. [Install OpenJDK 8](#step-2-install-openjdk-8) (a prerequisite for installing FAS)
3. [Install FAS](#step-3-install-fas)
4. [Create an image of your FAS-OpenJDK container](#step-4-create-an-image-of-your-fas-openjdk-container)
5. [Access FAS via your web browser](#step-5-access-fas-via-your-web-browser)

### Extra: Differences between a server/VM & a container
A container is like a mini VM, separated from your computer. It has its own Operating System (OS) - e.g. Rocky/ RHEL - and its own filesystem (folders, programs etc.) separate from yours. A big difference is that a container is lightweight - excluding a lot of tools you'd normally see on a full-blown Linux server/VM. But you can usually install them.
- We will have to install certain common packages from scratch, like `wget` and `service`. Common tools/commands like `ps` or `top` might also not be installed, but alternatives may be available if you don't want to install them.
- We won't need to disable SELINUX (Security-enabled Linux) as we would on a server/VM
- Firewalld is also not installed on a container, so no need to disable that either

## Prerequisites
Install Podman (on WSL Ubuntu)
- `sudo apt-get update`
- `sudo apt-get -y install podman`
- For other OSs, see [here](https://podman.io/docs/installation).

## Step 1: Run a container on WSL with the Rocky 8 image
1. Create a fresh container: `podman run -it -d rockylinux:8`.
    - `-d` runs the container in the background
2. You can see the running container and its ID with `podman ps`
3. Connect to the running container with `podman exec -it <containerID> sh` (use `exit` to get back to your WSL terminal)
4. Install `wget`: `dnf install -y wget`
5. Also install `service`: `dnf install -y initscripts`

### Extra: Why aren't we running a pure RHEL 8 container?
RHEL stands for "Red Hat *Enterprise* Linux", hence you need to pay for a subscription. Rocky is built from the same source code as RHEL, minus Red Hat branding and support. Nevertheless, this is how you can run a RHEL 8 container, if you want:

1. Create a personal account on [RHEL's site](https://sso.redhat.com/auth/realms/redhat-external/login-actions/registration?client_id=customer-portal&tab_id=dT_ziE4o2SM&client_data=eyJydSI6Imh0dHBzOi8vYWNjZXNzLnJlZGhhdC5jb20vc2VydmljZXMvcHJpbWVyL3Nlc3Npb24vc2NyaWJlLz9yZWRpcmVjdFRvPWh0dHBzJTNBJTJGJTJGYWNjZXNzLnJlZGhhdC5jb20lMkYiLCJydCI6ImNvZGUiLCJybSI6ImZyYWdtZW50Iiwic3QiOiIxYmYzOGEzNi05ZDJkLTQ3YTMtODc1MS1mMGE1Y2FlYWQ2MzAifQ)
2. On WSL, log into Red Hat's container image registry:  `podman login registry.redhat.io` (you'll be prompted to enter the username and password you made above)
3. Try creating a container with a RHEL 8 image: `podman run -it registry.redhat.io/rhel8/rhel-8:8.9`
        **NB:** If you haven't paid for a subscription, you'll get an "unauthorized" message

## Step 2: Install OpenJDK 8
You can install the **latest** OpenJDK 8 update & build by simply using `su -c "yum install java-1.8.0-openjdk"`. Then run `java -version` to confirm a successful install.

If you want to run a specific update and build number, a few more steps are involved:
1. If you want a specific build, find it [here](https://github.com/adoptium/temurin8-binaries/releases), and copy the link ending with "x64_linux_hotspot_8<update+build>.tar.gz"
2. Run `wget -O /home/openjdk-8.tar.gz "https://github.com/adoptium/temurin8-binaries/releases/download/jdk8u442-b06/OpenJDK8U-jdk_x64_linux_hotspot_8u442b06.tar.gz"` (replace the https URL with the one you copied)
3. Extract the tar file to the 3rd-party/"optional" directory: `tar -xzf openjdk-8.tar.gz -C /opt/`
4. Add a symlink for ease: `ln -s /opt/<extracted-folder> /opt/java`
5. Add the Java `bin` to your `PATH`: `export PATH=/opt/java/bin:$PATH`
6. Verify: `java -version`
   
## Step 3: Install FAS
1. Create the installer & fusion directories: `mkdir /opt/installer /opt/fusion`
2. Download the zip file from the [Reseller Portal](https://support.cba-japan.com/download/fas-2-6-1-1/). Unzip it and move the `fas-core-installer*` folder to your WSL's `/home/` directory.
3. Copy it to the container's filesystem: `podman cp /home/fas-core-installer* <containerID>:<path/inside/container>`
    - E.g. `podman cp ~/fas-core-installer-2.6.1-1 0149cd1bbcdd:/opt/installer/`
4. Change to the fas installer folder: `cd /opt/installer/fas-core-installer*`
5. Edit the `advanced-install.properties` file: `vi advanced-install.properties`
    - Set the `JDKPath` to `/opt/<jdkfolder>` (e.g. `JDKPath=/opt/jdk8u442-b06`)
    - NB: Since this is a container, set `bind.address.service` and `bind.address.management` to `0.0.0.0`
    - Save the file: `wq!`
6. Install FAS: `java -jar <fas-installer>.jar -options advanced-install.properties`
7. Verify that the 5 x FAS processes are running
    - You can either run `pgrep -a java | less` inside the container (`ps` isn't installed)
    - Or, you can `exit` the container, and run `podman ps <containerID>` to see the processes - - Either way, you should see:
        - `-D[Process Controller]`
        - `-D[Host Controller]`
        - `-D[Server:appserver-0149cd1bbcdd]`
        - `-D[Server:loadbalancer-0149cd1bbcdd]`
        - `-D[Server:management]`

## Step 4: Create an image of your FAS-OpenJDK container
If you were to remove your container now, all the work you've done to install OpenJDK and FAS will be lost. Hence, we're creating an image which we can reuse for future testing (like installing FCSDK)
1. Stop (don't remove) the container: `podman stop <containerID>`
2. Create an image of your container: `podman commit <containerID> <my-image-name>:<tag>`
    - E.g. `podman commit 0149cd1bbcdd my-image:my-tag`
    - This will be stored on your local image repository, and can be viewed with `podman image ls`
3. Now you can spin up a container with OpenJDK and FAS already installed! (see below)

## Step 5: Access FAS via your web browser
1. Run `podman run -it -d -p 9999:9990 --name my-rocky-container my-image:my-tag`
    - `-d`: Detached mode
    - `-p 9999:9990`: Port-forward the container's port (9999 as seen in the properties file you edited) to your local machine's 9990 port (i.e. 127.0.0.1:9990)
    - `--name`: A short name for the container, instead of using the container's ID each time
2. Exec into the container `podman exec -it my-rocky-container sh` and start fas: `service fas start`
3. Go to https://127.0.0.1:9990 on your browser, and enter the username and password from the properties file.

## Useful Commands
### Common Podman commands
- `podman start <containerID>`
- `podman stop <containerID>`
- `podman top <containerID>` to see the processes running inside the container
- `podman rm <containerID>` to delete the container

### Common FAS/MB commands
- `service fas status` (or `start`, `stop`, `restart` etc.)
