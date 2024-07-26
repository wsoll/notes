# Cheat sheet
```bash
docker ps -a
docker ps
docker images
docker volues ls

# run, enter, not exited(0)
docker run -it {image}:{version}
# detached, not exited(0)
docker run -td {image}:{version}
# run, pass env
docker run --env-file ./env.list {image-name}:{version}

#stop, kill all running containers
docker stop $(docker ps -a -q) &
docker kill $(docker ps -q)

# Enter
docker exec -it {container-id} bash

docker build -t {image-name}:{version} . 

# containers
docker rm -f $(docker ps -a -q)
# volumes
docker volume rm $(docker volume ls -q)
# images
docker rmi -f $(docker images -a -q)
# corrupted images
docker image prune -f

docker inspect <container-id> | grep {attribute}

docker-compose up -d {service}
docker-compose up
docker-compose -f {docker-compose.yml} up
docker-compose -f {docker-compose.yml} -f {service} down
docker-compose kill
```