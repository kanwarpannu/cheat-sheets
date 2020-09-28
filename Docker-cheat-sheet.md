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
`docker run -d -p 6379:6379 --name my-redis redis <image-name>`  

10. Pull a docker image from docker-hub:  
`docker pull <image-name>:<tag>`  

11. Build Dockerfile:  
`docker build -t <name-for-image> <dir-of-Dockerfile>`  

12. Mount a local directory(volume) for runtime (like to access property files, so you can change them on fly, only for dev):  
`docker run -p 80:80 -v <local-directory>:<where-to-access-in-container> <image-name>`  

13. Run docker commands with env variables:  
`docker run docker run -e "SPRING_PROFILES_ACTIVE=prod" -p 8080:8080 -t <image-name>`  
Or (For java debugging, might not work with Mac)  
`docker run -e "JAVA_TOOL_OPTIONS=-agentlib:jdwp=transport=dt_socket,address=5005,server=y,suspend=n" -p 8080:8080 -p 5005:5005 -t <image-name>`  

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
[Here](https://gist.github.com/kanwarpannu/5a3b4bbcd094a1ab299d39abbc5ca6e7) is a sample Dockerfile for java 8 projects.  
  
## Docker compose
It replaces the docker run command to make it easier to run.  
Command is: `docker-compose up -d` (d stands for the detached mode)  

If you want a fresh dockerfile build every time (important during dev) use:  
`docker-compose build --no-cache && docker-compose up -d`  

To stop all container in a docker-compose file: `docker-compose down` or `docker-compose down -v` (deletes all volumes)
  
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
  
[Here](https://gist.github.com/kanwarpannu/1db687901adc9d7ac969f5747fede6c7) is docker-compose.yml file for a project having 2 services.