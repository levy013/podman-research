## Podman R&D
https://uit1446:9090/podman

### Workflow
- Build image for app locally and push to remote server.
- In the future we can try leveraging auto updates for automatic rollouts on containers running specific images.

## Step 1: Install Podman Locally
Podman must be installed on the host machine where the application(s) are built, as well as remotely on the server where the containers will run: https://podman.io/

> I went with the CLI version since this workflow really only requires that we build the image locally

On Windows, Podman runs inside of a WSL2 container, but you can still communicate with the service via cmd or powershell. The installer should handle the setup + config of WSL if it's not already enabled on your machine.

📝 [Podman for Windows](https://github.com/containers/podman/blob/main/docs/tutorials/podman-for-windows.md)

## Step 2: Create a Dockerfile 
Here's an example dockerfile for `CAD.MQ.API`
> I let Rider create one for me and then I made modifications as necessary.

```dockerfile
# Set up .NET runtime env
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 8080

# Build Project
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src 
COPY ["/CAD.MQ.API/CAD.MQ.API.csproj", "CAD.MQ.API/"]
RUN dotnet restore "CAD.MQ.API/CAD.MQ.API.csproj"
COPY . .
WORKDIR "/src/CAD.MQ.API"
RUN dotnet build "./CAD.MQ.API.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./CAD.MQ.API.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .

ENTRYPOINT ["dotnet", "CAD.MQ.API.dll"]
```
### Super High-Level explanation of a dockerfile

Dockerfiles are powerful because they leverage a concept called `layered caching` to efficiently build images of an application. Without caching, we would need to do things like reinstall dependencies and recompile the source code every single time we rebuild an image 🫣

- A dockerfile is essentially just a chain of sequential commands called layers. e.g. FROM, COPY, RUN, etc.
- These layers are cached and evaluated sequentially.
- A layer's cache is invalidated if
    - the underlying command changes
    - any files it depends on changes
    - any prior layer has been invalidated, e.g. if we change the .NET runtime from aspnet:9.0 to aspnet:10.0, we would essentially need to rebuild the entire image
- `.dockerignore` files can help with preventing unneseccary cache invalidation due to changes in dotfiles like .git or changes in bin/ and obj/
 
**Example:**

❌ Bad
```dockerfile
COPY . .
RUN dotnet restore #runs even if we simply added a comment or removed whitespace 
```

✔️ Good 
```dockerfile
COPY project.csproj .
RUN dotnet restore #only runs if the .csproj changes 

COPY . .
RUN dotnet build #only runs if the source code changes
```
*... there's some nuance here, but you get the idea*

📝[Caching Dockerfiles](https://docs.docker.com/build/cache/)

## Step 3: Push Image to Remote Server
1. Open terminal and cd to `CAD24x7`
2. Build Image: 
```bash
podman build -f CAD.MQ.API/Dockerfile -t cmqapi:latest .
```
> -f: specify path to dockerfile
>
> -t: tag name

3. Save image as .tar:
```bash
podman save -o cmqapi.tar localhost/cmqapi:latest`
```
> `localhost` is automatically prepended in the previous step because we're not specifying a registry or a namespace
>
> This is Podmans way of indicating that this is a local image and has not been pulled or pushed to a remote repository

4. Push to uit1446:
```bash
scp cmqapi.tar lev013@uit1446.govt.hcg.local:/home/lev013`
```

## Step 4: Start App in Container
1. Open terminal and cd to dir containing image .tar
2. Load image from .tar:
```bash
podman load -i cmqapi.tar`
```
3. Start container
```bash
podman run -d -p 8080:8080 --name cmqapi localhost/cmqapi:latest
```
> -d: run detached
> 
> -p: fwd port on host to port of container
> 
> --name: name of container


### Considerations
1. Generalized CAD user to manage apps?
2. Are images automatically refreshed when we load a new one
3. Can we compile all of these commands into a rollout script on local? Can we set up a listener in systemd on uit1446 to respond to the new image and load it and auto update?
4. Better place to save .tar? right now it's just dropped into CAD24x7 so we need to remove it. Win doesn't do like a /tmp dir where files are auto cleared. But maybe we don't want that? Maybe we do want to add a directory somewhere to stored compressed images like this.


### TODOs 
1. We're hardcoding the values in Config.cs as opposed to reading from CADConfig.json - need to fix this. Without hardcoding, the server had a hard time looking up the values for the key(s) 

