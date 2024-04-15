# Continuous delivery (CD) workshop

This workshop aims students of EPITA (promotion 2025) of the SIGL specialisation in the context of the course of SOCRA.

**Objective**: Add continuous delivery (CD) to Sotracteur.

In this workshop, you will learn:

- how to dockerize a simple web page (a.k.a frontend)
- how to implement CD using Github Actions

## Requirements: push your web page

If it's not done yet, please push your `index.html` file from previous workshop to `main` branch of **your** `sotracteur` git repository.

## Step 1: Dockerize your application

**Objectif**: Use docker to run your `nginx` + static `html` page inside a Docker container.

- Create a Dockerfile at the root of your repository
  - As a base image, use [nginx:1.25](https://hub.docker.com/_/nginx) from docker hub.
  - Copy your `index.html` to the location of the default `index.html` file of
    nginx: `/usr/share/nginx/html/index.html`
- Try to build and run your page on your personal computer:
  - `docker image build -t sotracteur .`
  - `docker container run --rm -p 8080:80 sotracteur`
  - You should see your application running on http://localhost:8080 from your
    browser

Once working, push your Dockerfile on `main`.

## Step 2: Run your container on the remote server

We made some changes to your VM. We installed Traefik as a web server / reverse
proxy.
This means, you will **NOT** need to install any web server on your VM.

Here is a simple overview of your production:
![sotracteur production](images/sotracteur-production.png)

> Note: Traefik is a web server that fits perfectly with containerized applications. For
> our use case, we have already installed and configure Traefik on your VM.
>
> Traefik inspects all running container from the same docker network for docker
> labels. Based on those labels, it will configure for you the reverse proxy to
> route traefik to your application, based on the host name of the incomming HTTP request.
> Further reading here: [docs.traefik.io/traefik/](https://docs.traefik.io/traefik/)

### How to transfer your docker images from your personal computer to Scaleway's VM?

- You are building your docker image on your personal computer
- You need to run your docker image on your production's machine (Scaleway's VM)

You need an external place to store/get our images since we have to send our image among several environment.

For this, you will use the [docker container registry provided by Github](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry).

Next, from your personal computer, you will:

- `push` your docker image using github registry
- `run` your docker image from your production VM

### Push your docker image using Github's container registry

**Objective**: push sotracteur docker image to the github container registry (GHCR)

#### Create a private access token (PAT)

In order to use Github registry, a tool that will keep the docker images for you, you need to set your github project.

- Create a Github private access token (PAT) with read/write access on registry.
- From the personal account of one of the members (only 1 team member needs to do it):
  - go to your profile > settings > developer settings
  - Personal access tokens > Tokens(classic) > Generate new token (classic):
    - Note: sotracteur
    - Expiration: January 2025
    - Selected scopes:
      - `write:packages`
      - `delete:packages`

**Keep your key somewhere safe. You won't see it again.**

Store your PAT in a file that you will **NOT** push on your github repository.

> Note: anyone having access to this token can potentially interact with your repository, write and delete packages (see. selected scope). So you don't push this secret anywhere

#### Use your PAT to push a docker image to github's container registry

From your personal computer, you need to first login to this registry.

```sh
cat <path_to_your_PAT_file> | docker login ghcr.io -u <GITHUB_USERNAME> --password-stdin
```

You should see `Login Succeeded`

To be able to push a docker image to a registry, you need to have the correct name format for your docker image.

This means, you will need to build your docker image with a specific tag like:

```sh
docker image build -t <registry_name>/<github_username>/<repository_name>/<image_name>:<image_tag> .
```

- The `registry_name` needs to be `ghcr.io`
- The `github_username` needs to be the one who created the repository
- The `repository_name` will be the name of your project on github, it should be `sotracteur`
- The `image_name` could be `sotracteur-frontend`, since this is the name of the app, but you are free to chose any valid image name.
- The `image_tag` could be `v1` since it's the first version of Sotracteur. Once again, you're free to use any valid image tag.

Once it is build, to publish your image, you just have to push the freshly created image using `docker image push` command:

```sh
  docker image push <registry_name>/<github_username>/<repository_name>/<image_name>:<image_tag>
```

> Note: you need to push from the same session where you typed `docker login`.

### Run your docker image from your production VM

From your Scaleway VM (`ssh`):

- Login to the Github container registry, same way you did on your personal computer

> Note: you can use the `docker login ghcr.io -u ... -p <PAT>` instead of `cat <PAT> | docker login ghcr.io -u ... --password-stdin` to avoid `scp` your PAT to your machine. But both are fine.

- Pull your docker image using `docker image pull` command:

```shell
docker image pull <registry_name>/<github_username>/<repository_name>/<image_name>:<image_tag>
```

- When running your container, you need to specify Traefik labels:

```sh
"traefik.http.routers.sotracteur.rule=Host(\`groupXX.socra-sigl.fr\`)"  # determines which domain name Traefik will reverse proxy to
"traefik.http.routers.sotracteur.tls=true" # Activate TSL to allow HTTPS protocol over HTTP (we setup automatic SSL for you)
"traefik.http.routers.sotracteur.tls.certresolver=myresolver"  # Sets how to resolve SSL certificates
"traefik.enable=true" # Allow traefik to inspect the container's label
"traefik.docker.network=web" # Explicitly tells traefik to inspect docker network named web (from your VM; you can see it via `docker network ls`)
```

- You also need to make sure you run your container in the same `docker network` as Traefik: `web`
- And add a `name` to recognize your running container: `--name sotracteur-frontend`

So your command to run the container will look like:

```sh
  # from your VM; and adapt groupXX by your group number
  docker container run -d --network web \
    --name sotracteur-frontend \
    --label "traefik.http.routers.sotracteur.rule=Host(\`groupXX.socra-sigl.fr\`)" \
    --label "traefik.http.routers.sotracteur.tls=true" \
    --label "traefik.http.routers.sotracteur.tls.certresolver=myresolver" \
    --label "traefik.enable=true" \
    --label "traefik.docker.network=web" \
    <docker_image_name>
```

> Note: <docker_image_name> being your <registry_name>/<github_username>/<repository_name>/<image_name>:<image_tag>
> Note 2: escaping \`groupXX.socra-sigl.fr\` is necessary since you will run docker command on a shell;
> it would interpret groupXX.socra-sigl.fr as a shell command otherwise

You should be able to see your page live once again at the address: https://groupXX.socra-sigl.fr

If so, you've just your web application using containers, congrats!

### Step 3: Automate the deployment process using Github actions

This is the fun part of the workshop, where you will automate the whole process!

**Objective**: on **every commit to `main` branch**, I want my application to be **deploy on the production VM**.

In other words, we want you to implement continuous delivery for Sotracteur.

What you've done sofar:

- build your application using docker
- publish your application using docker and github container registry (ghcr.io)
- deploy your application using `ssh` and `docker run ...` command

#### Step 3.1: Create your first Github workflow

Github action is a CD tool that is fully integrated to github.

To integrate Github Actions to your project, you just have to create a new `yml`
file in your project under `.github/workflows/` directory.

Fortunately, Github provides an easy way to create a template file to avoid any
lost of time on typos.

From your Github project page:

- Select the `Action` tab > click `configure` on the `Simple workflow` template
- Remove the line

```yaml
pull_request: [main]
```

- Commit the file clicking the `Start commit` green button on main, leaving all
  defaults values. Then click `Commit new file`
  - Move to the `Actions` tab of your github's repository page
  - You should see your first workflow's first run

What just happened:

- you've setup a workflow that will be triggered on every `git push` on `main`
- the triggered workflow will run on **Github's servers**; with an image of `ubuntu`
- you can check [all available base OS your can use with github action](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners#supported-runners-and-hardware-resources)

#### Step 3.2: Configuring your workflow

Before you start, make sure you:

- `git pull origin main` on your local host after commiting your workflow
- understood the previous step
- Read again this `deploy.yml` file with comments to understand the structure.

To roughly sum up, a workflow is composed of:

- a name (to identify your workflow in the `Actions` tab on Github), specified
  by the `name: ...` line
- a job trigger (here, on every commit to main), specified by the `on: ...` lines
- `jobs` with a set of `steps`:
  - 1 job is composed of 1 or more steps. A step could be a `shell` command or
    a github action. This is the atomic element of your workflow.
  - 1 job `runs-on` a machine with a specific system. We will use
    `ubuntu-latest`

##### Let's configure our workflow.

Before starting building steps, we will add several secret variables.

> Note: We use Github secrets variable because you don't put your secret API
> keys directly in your code. If you share it by mistake, the owner of this key
> can control all your registry.

You can add any `key=value` secrets in: Your Github project > Settings > Secrets and variables > Secrets > New repository secret

You will add those secret variables:

```yml
DOCKER_USER=<your_github_username>
DOCKER_PASSWORD=<your_github_PAT>
SSH_HOST=groupXX.socra-sigl.fr
SSH_USER=sigl
SSH_KEY=<private_key_from_previous_workshop>
```

Now that we are all set, we have modify our workflow to:

- build the docker image
- publish the docker image
- run our docker image (on our production VM)

##### Build step

From your local machine, you used `docker image build` command.

Fortunately, the `ubuntu-latest` system on which our `steps` are running do have
docker installed and git by default.

But the machine doesn't have our project installed on it.

So we will leave the first step that uses `actions/checkout@v2`. This checks out
the project in this VM machine on ubuntu.

Just add the build step:

```yml
- name: build docker image
  run: docker image build -t <docker_image_name> .
```

> Note: remeber the format you should respect for your `docker_image_name` !

Try out your workflow by commiting your changes.

##### Publish step

You will use your github docker registry to push/pull your docker image.

Add to your `name: build docker image step`, commands to:

- login to your github docker registry
- push your images (careful with the image format!)

Now, you are not entering your Github PAT (private access token) directly in the yaml file.

Instead, you can refer your token by using secrets variables like:

```yml
# you've set it on your project earlier. This will be substitute with the correct
# secret value when the job will be executed.
${{ secrets.DOCKER_PASSWORD }}
```

For example, if you would (and you shouldn't!) print your secret variable, you would write:

```yml
- name: Do NOT do this at home!
  run: |
    echo "My secret docker user is: ${{ github.DOCKER_USER }}"
```

So you will use your docker secrets variables when using `docker login` inside your `deploy.yml` workflow.

For the `<image_tag>`, you will use the `${{ github.sha }}` default variable. This will make sure that our image are unique from each git commit. It makes it easy to trace which version of the code is in which image.
Try out again to commit/push your changes, and your workflow to be green.

##### Deploy step

You are a the last step of finishing your continuous delivery pipeline!

To deploy, we need to:

- Connect remotely to your produciton VM
- Pull and run your docker image from your production VM

To use `ssh` in your workflow, you will use the existing github action [appleboy/ssh-action](https://github.com/appleboy/ssh-action)

Here is how you should use this action for this workshop:

```yml
# - ...
# As as step, same structure as checkout github action,
# with a bit more options
- name: Deploy on production VM
  uses: appleboy/ssh-action@master
  env:
    TAG: ${{ github.sha }}
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    envs: TAG
    script: |
      echo "image tag to be release: $TAG"
      echo "TODO: login, pull and run your docker image :)"
```

You see that we are using the secrets we've entered during previous steps

Then, you should repeat the same commands you've done in when you manually deployed your container earlier:

- Login to your personal github docker registry
- Pull your `<docker_images_name>` with the specific `$TAG`
- Run your container with the correct `web` docker network and the several docker labels for Traefik

> WARNING: You cannot use line breaks like on a regular shell using `\` + `newline` for a single docker command. So you will have to put all the `docker run ...` in one line without `\` + `newline`...

It will work if you run it once, but second time, you must stop the previous container. And for the workflow not to fail when nothing is running, we can use a little `shell` trick: `exit 1 || echo "this has failed, but not the whole command"`

In order to have a nice functionnal CD that works on every run, you should add this line before running your container:

```yaml
# ...
script: |
  # ...
  # If sotracteur-frontend is your container name
  docker container stop sotracteur-frontend && docker rm sotracteur-frontend || echo "Nothing to stop"
  # your docker run command from previous step. Use ${{ github.sha }} as an image tag
  docker container run -d --name sotracteur-frontend --network web --label ...
```

Once this is done, try out to push different versions of your `index.html` on main and this should continously deliver the new version on `https://groupXX.socra-sigl.fr` !

### Challenge: Manual deployment trigger

**Objective**: Offer the possibility to run your workflow manually from the UI of Github

You will use the [`workflow_dispatch` functionality](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch)

As input, you will give the docker image version you wish to deploy.
