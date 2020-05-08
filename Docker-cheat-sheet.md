# List of common docker commands

1. List all containers:  
`docker ps -a`  

2. Stop a container:  
`docker stop <container-name>`  

3. Remove a container:  
`docker rm <container-name>`  

4. Start a container:  
`docker start <container-name>`  

5. Check all downloaded images:  
`docker image ls`  

6. Remove an image:  
`docker image rm <image-id>`  

7. Check logs:  
`docker logs <container-name>`  

8. Run shell inside docker container (here assuming bash, and linux containers):  
`docker exec -it <container-name> bash`  

9. Run a container in the detached mode (Here starting redis on port 6379):  
`docker run -d -p 6379:6379 --name my-redis redis`  

10. Pull a docker image from docker-hub:  
`docker pull <image-name>:<tag>`  

11. Build Dockerfile:  
`docker build -t <name-for-image> <dir-of-Dockerfile>`  

12. Mount a local directory(volume) for runtime (like to access property files, so you can change them on fly, only for dev):  
`docker run -p 80:80 -v <local-directory>:<where-to-access-in-container> <image-name>`  

## Dockerfile
Dockerfile is built into an Image, which when run, becomes container.  
Ideally we want one process per service as a container lasts one lifetime of a service. 

A standard Dockerfile structure:  
```
FROM <base-image-name>:<tag>
COPY <location-file-to-copy> <where-to-copy-in-container>
CMD ["<cmd-to-run>", "argument1"]
EXPOSE <port-to-expose>
```  
When using an image from docker-hub look at read-me to see the structure of Dockerfile, and steps on how to run it.

## Example of Dockerfile for springboot project
## Docker compose
It replaces the docker run command to make it easier: `docker-compose up`  

A standard docker-compose.yml file:  
```
version: '<latest-docker-compose-version>'

services:             //all services here
  service-one:    //name of service we want to give it
    build: <dir-of-dockerfile>      // what we want to build
    volumes:
      - <local-dir>:<dir-in-container>   //this is a list of volume we want to mount(for dynamic code change), only for dev
    ports:
      - <local-port-no>:<container:port>   //local needs to be exposed in docker file

  service-two:
    image: <docker-image>:<tag>
    volumes:
      - <local-dir>:<dir-in-container>
    ports:
      - <local-port-no>:<container:port>
    depends_on:
      - <service-one>    //services, if, its dependent upon
```