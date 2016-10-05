# IBM dashDB Local Offline Installation

There are essentially two steps required for an offline installation:

1.  Downloading or building the Docker bundle
1.  Preparing a server for deploying IBM dashDB Local

The Docker bundle file (`docker-offline.tar`) contains all system packages required for installing Docker, as well as the IBM dashDB Local Docker image. Its size is approximately 3 GB, and it takes approximately 30 minutes to build it, depending on your build server specs.

## Prerequisites

* **[Required if building]** Access to [https://hub.docker.com/r/ibmdashdb/local/](https://hub.docker.com/r/ibmdashdb/local/).

* **[Required if building]** Build server
  * RHEL 7
  * 3.10+ kernel
  * internet access
  * admin access

* Lab server
  * RHEL 7
  * 3.10+ kernel
  * no internet access
  * admin access

## Obtain Docker Bundle

If you chose to download the Docker bundler, skip to the [Prepare Lab Server](#prepare-lab-server) section.

### Prepare the Build Server

1.  Update system and install dependencies. (ETD: ~3 min)
    ```shell
    sudo yum update -y
    sudo yum install -y yum-utils createrepo
    ```

1.  Install and start Docker Engine.

    ```shell
    sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

    sudo yum install -y docker-engine
    sudo systemctl enable docker.service
    sudo systemctl start docker
    ```

1.  Test Docker Engine.

    ```shell
    sudo docker run --rm hello-world
    ```

1.  Download IBM dashDB Local image. (ETD: 5 min)

    ```shell
    # This is an interactive command; it'll prompt for your Docker ID username and password
    sudo docker login

    sudo docker pull ibmdashdb/local:latest-linux
    ```


### Create the Docker Bundle

1.  Create work directory.

    ```shell
    mkdir -p docker-offline/{docker,images}
    ```

1.  Download Docker Engine's package and its dependencies. (ETD: 15 min)

    ```shell
    sudo repoquery --tree-requires --resolve $(sudo rpm -qa) | \
      sed -E 's/(\[.*\]|\||\s|\\\_)//g' | \
      sort | uniq > docker-offline/docker/.packages

    cat docker-offline/docker/.packages | xargs yumdownloader --destdir docker-offline/docker
    ```

1.  Create local Yum repository.

    ```shell
    createrepo --database docker-offline/docker
    ```

1.  Save Docker images to files. (ETD: 15 min)

    ```shell
    sudo docker save hello-world:latest | \
      gzip -9 > docker-offline/images/hello-world__latest.tgz

    sudo docker save ibmdashdb/local:latest-linux | \
      gzip -9 > docker-offline/images/ibmdashdb_local__latest_linux.tgz
    ```

1.  Create the Docker bundle file.

    ```shell
    tar cvf docker-offline.tar docker-offline/
    ```

1.  Finally, transfer `docker-offline.tar` over to the lab server.


## Prepare the Lab Server

1.  Extract files from bundle.

    ```shell
    tar xvf docker-offline.tar
    ```

1.  Setup Yum repository.

    ```shell
    mkdir -p /var/local/yum
    mv docker-offline/docker /var/local/yum

    sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[docker]
name=Docker
baseurl=file:///var/local/yum/docker
enabled=1
gpgcheck=0
EOF
    ```

1.  Install Docker Engine.

    ```shell
    sudo yum install -y docker-engine
    sudo systemctl enable docker.service
    sudo systemctl start docker
    ```

1.  Test Docker Engine.

    ```shell
    docker load -i docker-offline/images/hello-world__latest.tgz
    docker run --rm hello-world
    ```

1.  Load IBM dashDB Local image. (ETD: 3 min)

    ```shell
    docker load -i docker-offline/images/ibmdashdb_local__latest_linux.tgz
    ```


## Run IBM dashDB Local

Follow the instructions below. You can find detailed instructions at [https://hub.docker.com/r/ibmdashdb/local/](https://hub.docker.com/r/ibmdashdb/local/).

1.  Create the data directory and the `options` file.

    ```shell
    mkdir -p /mnt/clusterfs

    touch /mnt/clusterfs/options
    sed -i.$(date +%s).bkp -E "/^ENABLE_ORACLE_COMPATIBILITY/d" /mnt/clusterfs/options
    echo "ENABLE_ORACLE_COMPATIBILITY='YES'" >> /mnt/clusterfs/options

    ```

1.  Deploy IBM dashDB Local and tail its log.

    ```shell
    docker run -d -it \
      --privileged=true \
      --net=host \
      --name=dashDB \
      -v /mnt/clusterfs:/mnt/bludata0 \
      -v /mnt/clusterfs:/mnt/blumeta0 \
      ibmdashdb/local:latest-linux \

    docker logs -f dashDB
    ```

1.  After several minutes, the following message will be logged.

    ```
    ***********************************************************
    *******             Congratulations!             **********
    **         You have successfully deployed dashDB         **
    ***********************************************************
    ```

1.  Hit `ctrl-c` to exit the log and run the command below to change your password.

    ```shell
    docker exec -it dashDB setpass <NEW_PASSWORD>
    ```

1.  IBM dashDB Local is ready for use.

    ```
    URL:      https://<LAB_SERVER_IP_ADDRESS>:8443
    username: bluadmin
    password: <NEW_PASSWORD>
    ```
