# IBM dashDB Local Offline Installation

## Prerequisites

* **[Optional]** Access to [https://hub.docker.com/r/ibmdashdb/local/](https://hub.docker.com/r/ibmdashdb/local/).

* **[Optional]** Build server - used to build the Docker bundle
  * RHEL 7
  * 3.10+ kernel
  * internet access
  * admin access

* **[Required]** Lab server - runs IBM dashDB Local
  * RHEL 7
  * 3.10+ kernel
  * no internet access
  * admin access


## Prepare Docker Bundle

### Prepare Build Server

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


### Create Docker Bundle

1.  Create work directory.

    ```shell
    mkdir -p docker-offline/{docker,images}
    ```

1.  Download Docker Engine's package and its dependencies. (ETD: 15 min)

    ```shell
    sudo repoquery --tree-requires --resolve $(sudo rpm -qa) \
      | sed -E 's/(\[.*\]|\||\s|\\\_)//g' \
      | sort | uniq > docker-offline/docker/.packages

    cat docker-offline/docker/.packages | xargs yumdownloader --destdir docker-offline/docker
    ```

1.  Create Yum repository.

    ```shell
    createrepo --database docker-offline/docker
    ```

1.  Save Docker images to files. (ETD: 15 min)

    ```shell
    sudo docker save hello-world:latest | gzip -9 > docker-offline/images/hello-world__latest.tgz
    sudo docker save ibmdashdb/local:latest-linux | gzip -9 > docker-offline/images/ibmdashdb_local__latest_linux.tgz
    ```

1.  Create bundle.

    ```shell
    tar cvf docker-offline.tar docker-offline/
    ```

1.  Finally, transfer the 'docker-offline.tar' file over to the lab server.


## Prepare Lab Server

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
    ## sudo rpm -Uvh --replacefiles docker-offline/docker/*.rpm
    ## sudo yum install -y docker-offline/docker/*.rpm


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

```shell
mkdir -p /mnt/clusterfs

touch /mnt/clusterfs/options
sed -i.$(date +%s).bkp -E "/^ENABLE_ORACLE_COMPATIBILITY/d" /mnt/clusterfs/options
echo "ENABLE_ORACLE_COMPATIBILITY='YES'" >> /mnt/clusterfs/options

docker run -d -it \
  --privileged=true \
  --net=host \
  --name=dashDB \
  -v /mnt/clusterfs:/mnt/bludata0 \
  -v /mnt/clusterfs:/mnt/blumeta0 \
  ibmdashdb/local:latest-linux \
date +%s
docker logs -f dashDB
```
