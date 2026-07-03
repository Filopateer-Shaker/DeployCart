# How To Those files?

## To Run The Containers
```bash
    ##### First You Have To Build the Image ##############

docker build -t <TagName> --progress=plain <PathToDockerfile>

    ##### How To Run The Container ##############

docker container run -p <ContainerPort:hostPort> -d -it --name <ContainerName> <Image>

    ##### How To Run The Docker-Compose File ##############

docker compose up < PathToTheDockerCompose > 

```

