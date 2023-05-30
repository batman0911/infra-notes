# Docker installation

## Set up the repository

- Update the apt package index and install packages to allow apt to use a repository over HTTPS:

```shell
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
```

- Add Docker’s official GPG key:
  
```shell
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

- Use the following command to set up the repository:

```shell
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## Install Docker Engine

- Update the apt package index:

```shell
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- Verify that the Docker Engine installation is successful by running the hello-world image.

```shell
sudo docker run hello-world
```

## Fix permission denied

- Create the docker group if it does not exist
  
```shell
sudo groupadd docker
```

- Add your user to the docker group.

```shell
sudo usermod -aG docker $USER
```

- Log in to the new docker group (to avoid having to log out / log in again; but if not enough, try to reboot):

```shell
newgrp docker
```

- Check if docker can be run without root

```shell
docker run hello-world
```