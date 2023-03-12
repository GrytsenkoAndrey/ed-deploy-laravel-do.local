# ed-deploy-laravel-do.local
How To Deploy A Laravel Project To DigitalOcean Droplet Using Github Action


## Prerequisites

To follow along in the article you should have a Laravel project already built and pushed to your GitHub repository, a basic understanding of GitHub Actions, and a digital-ocean account required. You can get started with Laravel here, and get started on Digital-ocean here. You will also need to create a droplet and set up your droplet, the following articles below can get you started.
1. [Create a droplet](https://docs.digitalocean.com/products/droplets/how-to/create/)
2. [Initial setup for a droplet](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04)
3. [Install the LEMP stack on the droplet](https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mysql-php-lemp-stack-on-ubuntu-22-04)

## Digital Ocean

Digital Ocean is a unique cloud hosting provider that offers cloud computing services to business entities so that they can scale themselves by deploying DigitalOcean applications that run parallel across multiple cloud servers without compromising on performance!

## Github Action

GitHub Actions is a continuous integration and continuous delivery (CI/CD) platform that allows you to automate your build, test, and deployment pipeline.
You can create workflows that build and test every pull request to your repository or deploy merged pull requests to production. GitHub Actions goes beyond just DevOps and lets you run workflows when other events happen in your repository.
For example, you can run a workflow to automatically add the appropriate labels whenever someone creates a new issue in your repository. GitHub provides Linux, Windows, and macOS virtual machines to run your workflows, or you can host self-hosted runners in your own data center or cloud infrastructure.

## Overview of GitHub Action Workflow

GitHub Action workflow consists of one or multiple jobs which contain one or multiple steps. To deploy our application, we need to create the following jobs:

## Create GitHub Action build artifacts for deployment

We want to achieve the following for our Laravel application artifacts:
1. Install NPM dependencies.
2. Compile CSS and Javascript assets.
3. Install Composer dependencies.
4. Archive our build and remove unnecessary data (e.g., node_modules).
5. Store our archive so we can deploy it to our servers.

## Prepare release on all our servers

We want the release preparation job to do the following:
1. Ensure we have a directory that holds every release.
2. Ensure we have a storage directory that shares data between releases.
3. Ensure we have a current directory that links to the active release.
4. Extract our build files into our releases directory.

## Run optional before hooks

This is an optional feature, but I want to execute specific commands before the release is activated (e.g., chmod directories). So there needs to be a way to configure these so-called before hooks.

## Activate the release

Now we are ready to activate our new release without any downtime. We can do this by changing symbolic links, this essentially swaps the underlying release, which is linked to our current directory, to a new release inside our releases directory. I'm running PHP FPM on my servers, so I also want to reload PHP FPM to detect the changes.

## Run optional after hooks

This is an optional feature as well, but I want to execute specific commands after the release is activated to send a notification that my deployment is completed for example. So there needs to be a way to configure these so-called after hooks.

## Cleaning up

Given that we are uploading and extracting new releases, we take up more disk space after each release. To make sure we don’t end up with thousands of releases and a full disk, we need to limit the number of release artifacts living on every server.

## Project scaffolding

To start you will need a GitHub repository as testing your workflow requires you to commit and push your workflow .yml file, you should have one already with your Laravel project, I recommend you create a separate branch for this, Be sure to verify that in your project everything works before you continue.
Lastly, from your project root directory create a new workflow file, feel free to give it any name you would like and place it in the _.github/workflows._

```
name: Deploy Application

on:
  push:
    branches: [ master ]

jobs:
  # Magic
```

Inside the github/workflows/any_name_you_like.yml file write the following, make sure you choose a different branch if you don’t want to clutter your commit history, as you might end up pushing to GitHub several times, as you follow along.

## Create deployment Artifacts

let’s kick off by checking out our project by using the predefined checkout action by GitHub. Before we have something to deploy, we will start the build our Laravel application as we would typically do.

```
# // code from earlier is ommited for clearity

jobs:
  create-deployment-artifacts:
    name: Create deployment artifacts
    runs-on: ubuntu-latest

  steps:
  - uses: actions/checkout@v3
  ```
  
  GitHub will checkout the code from our repository in the container; no further steps are necessary. Now that we have our code, we can continue compiling our assets.

```
# // code from earlier is ommited for clearity
 
- name: Compile CSS and Javascript
    run: |
      npm install
      npm run prod
```

_Tip: If you use a CDN to deliver your static files, be sure to implement your own solution. The front-end assets are now shared amongst all servers individually, which is not ideal. It may impact your website speed since assets could be loaded from different servers on every request depending on your load-balancer._

We can now continue with our back-end code. Before we can install any composer packages, we need to make sure PHP is installed first. We will use the `setup-php` action by [Shivam Mathur](https://github.com/shivammathur/setup-php), which makes this a breeze.
  
  ```
  # // code from earlier is ommited for clearity

  - name: Compile CSS and Javascript
    run: |
      npm ci
      npm run prod

  - name: Configure PHP 8.1
    uses: shivammathur/setup-php@master
    with:
      php-version: 8.1
      extensions: mbstring, ctype, fileinfo, openssl, PDO, bcmath, json, tokenizer, xml
  ```
  
  This will configure PHP 8.1 and install the required extensions. If your application requires additional extensions, be sure to add them to the list. We can continue by installing our Composer dependencies.

```
# // code from earlier is ommited for clearity

  - name: Composer install
    run: |
    composer install --no-dev --no-interaction --prefer-dist
```

Time to test out what we’ve got so far! Commit your code changes and push them to GitHub. Once you’ve done so, visit the Actions page (https://www.github.com/<username>/<repository>/actions)

You can click each job to see the execution output. If a job fails, it will show a red cross instead of a checkmark. The execution output will often provide you with the information you need to resolve the issue.

We’ve successfully compiled our front-end assets and installed our Composer dependencies. Now we need to store the results. GitHub provides an [Upload-Artifact](https://github.com/actions/upload-artifact) helper. This will help us to share the GitHub Actions artifacts between jobs.

You can upload single files, multiple files, and directories. Since we just want all our files deployed, I prefer to create a TAR archive, so we have a single file to work with. You could also create a ZIP archive, but this will require installing additional software in the build container as Ubuntu doesn’t ship with the required libraries.

```
 - name: Create deployment artifact
    run: tar -czf app.tar.gz *
```

This will create a new tar archive called **app.tar.gz** containing all the files, including the additional build artifacts we've made in the previous steps.

This works just fine, but the archive now contains files we don’t need, like the **node_modules** directory. We only required these to run the **npm run production** command. Let's fix this by excluding directories from our archive.

```
# //
- name: Create deployment artifact
  run: tar -czf app.tar.gz --exclude=*.git --exclude=node_modules --exclude=tests *
```
  
Our archive will now skip the **.git, node_modules,** and **tests** directories from our archive. If you have additional files that are not required to be on your production server, exclude them now. By making the archive smaller, your deployment will be quicker.

I want to change our archive's filename so it's easier to identify which commit our archive contains. GitHub has some global variables you can use in your .yml file, so let's change the name to the commit hash.

```
# //
- name: Create deployment artifact
  env:
    GITHUB_SHA: ${{ github.sha }}
  run: tar -czf "${GITHUB_SHA}".tar.gz --exclude=*.git --exclude=node_modules *
```

We use the **env** option to pass environment variables down into the container. In this case, we define an environment variable called **GITHUB_SHA** and assign it the commit hash from the current job. If you want to know more about context and expression syntax for GitHub Actions, click [here](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions).

```
# //
- name: Create deployment artifact
  env:
    GITHUB_SHA: ${{ github.sha }}
  run: tar -czf "${GITHUB_SHA}".tar.gz --exclude=*.git --exclude=node_modules *

- name: Store artifact for distribution
  uses: actions/upload-artifact@v3
  with:
    name: app-build
    path: ${{ github.sha }}.tar.gz
```

We only need to provide this step with the path to our file. We again use the GitHub Action expression to get the commit hash. Finally, we provide our artifact with a name, which we can use for later reference to download the GitHub Actions artifacts in our deployment job.

Before we can continue with any of the follow-up jobs, we need to prepare something called a GitHub Actions strategy matrix. A strategy creates a build matrix for your jobs. You can define different variations to run each job in.

You can define a matrix of different job configurations. A matrix allows you to create multiple jobs by performing variable substitution in a single job definition.

In our case, servers are variable in our jobs. We will use a JSON file inside our repository, so we don't have to switch back and forth between the GitHub UI.

Create a new file called deployment-config.json the root of your project and add the following contents:

```
[
  {
    "name": "server-1",
    "ip": "your_droplet_ip",
    "username": "your_droplet_username",
    "password":"your_droplet_password",
    "beforeHooks": "",
    "afterHooks": "",
    "path": "path_to_your_project"
  }
]
```

This file contains a list of all the servers we wish to deploy to. We can define the before and after hook, and our Nginx directory path, which will serve our application for each server. For authentication, we will use an SSH key which you can store inside a [repository secret](https://docs.github.com/en/actions/reference/encrypted-secrets). In this example, I will name the secret SSH_KEY and reference this secret inside our workflow to authenticate the commands we want to execute on our remote server.

You could even host this file somewhere to dynamically populate this file, so if you add more instances to your infrastructure, they will be automatically included in the deployment cycle.

_Note: A job matrix can generate a maximum of 256 jobs per workflow run. If you want to deploy to hundreds of servers in a single workflow, you need an alternative. I would recommend going serverless with a solution like Laravel Vapor._

To make this configuration available inside our GitHub Actions workflow, we need to export this data. Using a special syntax, GitHub can identify which data to assign to which variable to reference later on our workflow matrix.

```
# //
- name: Store artifact for distribution
  uses: actions/upload-artifact@v3
  with:
    name: app-build
    path: ${{ github.sha }}.tar.gz

- name: Export deployment matrix
  id: export-deployment-matrix
  run: |
      delimiter="$(openssl rand -hex 8)"
      JSON="$(cat ./.github/workflows/servers.json)"
      echo "DEPLOYMENT_MATRIX<<${delimiter}" >> "${GITHUB_OUTPUT}"
      echo "$JSON" >> "${GITHUB_OUTPUT}"
      echo "${delimiter}" >> "${GITHUB_OUTPUT}"
```

First, we need to get the JSON from our **deployment-config.json** by using a simple **cat** command. Next, we will use a delimiter to support muli-line contents like our JSON file. Finally, we append our JSON to the **${GITHUB_OUTPUT}** environment using **DEPLOYMENT_MATRIX** as our name/key.

If we want our other jobs to get the output from this job, we need to specify the output reference in our **create-deployment-artifacts** job.

```
# //
jobs:
  create-deployment-artifacts:
    name: Create deployment artifacts
    runs-on: ubuntu-latest
    outputs:
      DEPLOYMENT_MATRIX: ${{ steps.export-deployment-matrix.outputs.DEPLOYMENT_MATRIX }}
    steps:
      - uses: actions/checkout@v3

# //
```

We can now reference DEPLOYMENT_MATRIX other jobs and get all the required information for each of our servers.

## Prepare the release on all the servers
Next up, we can continue to prepare our release on every server. Thanks to the deployment matrix configuration we've created, we can cycle through each server to repeat all the steps on every server.

# //
prepare-release-on-servers:
  name: "${{ matrix.server.name }}: Prepare release"
  runs-on: ubuntu-latest
  needs: create-deployment-artifacts
  strategy:
    matrix:
      server: ${{ fromJson(needs.create-deployment-artifacts.outputs.DEPLOYMENT_MATRIX) }}
As you can see, there are a couple of new parameters. The first new parameter, needsallows us to make sure a specific step has finished before this job can start. In our case, we want to prepare the release on the servers once the create-deployment-artifacts. If you have multiple-step dependencies, you can also pass an array.

Next, we got our strategy parameter, which allows us to define our matrix. In this case, we define a matrix variable named server and assign it to our DEPLOYMENT_MATRIX we've created in our previous job. By default, the [build matrix](https://docs.github.com/en/free-pro-team@latest/actions/learn-github-actions/managing-complex-workflows#using-a-build-matrix) expects an array and not a JSON string.

As mentioned earlier, GitHub Actions [support context and expression syntax](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions) to access context information but also to evaluate expressions. This includes a couple of [functions](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#functions) to, for example, cast values. To read our server configuration, we need to change our JSON string into a real JSON object using the [from JSON](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#fromjson) function.

We can retrieve our JSON object through the matrix context. This enables access to the matrix parameters we've configured for the current job. For example, in the code above, you can see we define a variable job name: **name: "${{ matrix.server.name }}: Prepare release"**. In the GitHub UI, this will resolve to "server-1: Prepare release".

We are now ready to continue with the steps of our job, downloading our artifact to our build container, uploading our artifact to the server, extracting our archive, and setting up required directories on our server if they don't exist.

```
# //
prepare-release-on-servers:
  name: "${{ matrix.server.name }}: Prepare release"
  runs-on: ubuntu-latest
  needs: create-deployment-artifacts
  strategy:
    matrix:
      server: ${{ fromJson(needs.create-deployment-artifacts.outputs.DEPLOYMENT_MATRIX) }}
  steps:
    - uses: actions/download-artifact@v3
      with:
        name: app-build
    - name: Upload
      uses: appleboy/scp-action@master
      with:
        host: ${{ matrix.server.ip }}
        username: ${{ matrix.server.username }}
        key: ${{ secrets.SSH_KEY }}
        port: ${{ matrix.server.port }}
        source: ${{ github.sha }}.tar.gz
        target: ${{ matrix.server.path }}/artifacts
  ```
  
  We start by downloading our GitHub Actions build artifact, which is quite simple as GitHub provides a simple action out of the box called ["Download Artifacts."](https://github.com/actions/download-artifact) We reference the name we've used for uploading our artifact. This will download the artifact into the build container.

Next, we upload the artifact to the server using a third-party [SCP](https://github.com/appleboy/scp-action) action in the GitHub Actions marketplace. This action will copy our file via SSH based on our configuration. Make sure to check out the [repository](https://github.com/appleboy/scp-action) for all available input variables for this action if you, for example, want to use SSH keys for authentication.

The SSH credentials are self-explanatory; we simply reference our JSON object to get the server's connection details and the credentials. The SCP action requires a source input variable; this is the file we want to upload. We use the commit hash context object again to generate the filename
**${{ github.sha }}.tar.gz.**

Besides the source input variable, we also need to provide the target input variable. I want to keep all the uploaded artifacts in a separate directory. I reference the server path and append the path with the directory name artifacts to achieve this. The final path will be /var/www/html/artifacts.

Let's take a look and make sure everything is working so far. Commit all your changes and push them to GitHub. Visit the actions page of your repository again, and you should see a running action.

Click server-1: Prepare the release job so you can see the output of all the steps. Click on the Upload job to expand the output. Seems to be looking good so far, our GitHub Actions release artifacts are now uploaded to our remote server.

Our final step for this job is to extract our uploaded archive and create a couple of directories if they don't exist. We will use SSH commands to set this up. We will again use a third-party action in the GitHub Actions marketplace called SSH Action. By now, you will be familiar with most of the syntax. The input variables are similar to the previous upload step:

  ```
  # //
- name: Extract archive and create directories
  uses: appleboy/ssh-action@master
  env:
    GITHUB_SHA: ${{ github.sha }}
  with:
    host: ${{ matrix.server.ip }}
    username: ${{ matrix.server.username }}
    key: ${{ secrets.SSH_KEY }}
    port: ${{ matrix.server.port }}
    envs: GITHUB_SHA
    script: |
      mkdir -p "${{ matrix.server.path }}/releases/${GITHUB_SHA}"
      tar xzf ${{ matrix.server.path }}/artifacts/${GITHUB_SHA}.tar.gz -C "${{ matrix.server.path }}/releases/${GITHUB_SHA}"
      rm -rf ${{ matrix.server.path }}/releases/${GITHUB_SHA}/storage

      mkdir -p ${{ matrix.server.path }}/storage/{app,public,framework,logs}
      mkdir -p ${{ matrix.server.path }}/storage/framework/{cache,sessions,testing,views}
      chmod -R 0777 ${{ matrix.server.path }}/storage
  ```
  
  If you use the **|** character in your script, you can define multiple commands split across multiple lines you want to execute on your server.

```
# Create a new directory with the commit hash as the directory name
mkdir -p "${{ matrix.server.path }}/releases/${GITHUB_SHA}"

# Extract the tar file into our release directory
tar xzf ${{ matrix.server.path }}/artifacts/${GITHUB_SHA}.tar.gz -C "${{ matrix.server.path }}/releases/${GITHUB_SHA}"

# Create Laravel storage directories and set permissions
mkdir -p ${{ matrix.server.path }}/storage/{app,public,framework,logs}
mkdir -p ${{ matrix.server.path }}/storage/framework/{cache,sessions,testing,views}
chmod -R 0777 ${{ matrix.server.path }}/storage
```


If you want, you can SSH into one of your servers and verify that the directories exist and the archive is unarchived.

So far, so good! We've made our build, uploaded the results to our server, extracted the archive, and made sure the required directories exist. We are almost there


## Run before hooks

This step is optional, but it's quite useful if you want to execute specific commands before you activate your release. In this example, we are using Laravel, so you might want to run database migrations.
  
  ```
  # //
run-before-hooks:
  name: "${{ matrix.server.name }}: Before hook"
  runs-on: ubuntu-latest
  needs: [ create-deployment-artifacts, prepare-release-on-servers ]
  strategy:
    matrix:
      server: ${{ fromJson(needs.create-deployment-artifacts.outputs.DEPLOYMENT_MATRIX) }}
  steps:
  - name: Run before hooks
    uses: appleboy/ssh-action@master
    env:
      GITHUB_SHA: ${{ github.sha }}
      RELEASE_PATH: ${{ matrix.server.path }}/releases/${{ github.sha }}
      ACTIVE_RELEASE_PATH: ${{ matrix.server.path }}/current
      STORAGE_PATH: ${{ matrix.server.path }}/storage
      BASE_PATH: ${{ matrix.server.path }}
    with:
      host: ${{ matrix.server.ip }}
      username: ${{ matrix.server.username }}
      key: ${{ secrets.SSH_KEY }}
      port: ${{ matrix.server.port }}
      envs: envs: GITHUB_SHA,RELEASE_PATH,ACTIVE_RELEASE_PATH,STORAGE_PATH,BASE_PATH
      script: |
        ${{ matrix.server.beforeHooks }}
  ```
  
  Again, similar job but with a couple of changes. First, we want to make sure the create-deployment-artifacts and the prepare-release-on-servers have been completed by passing an array to the need property.

```needs: [ create-deployment-artifacts, prepare-release-on-servers ]```

To make things a bit easier when defining before hooks, I want to use specific environment variables to simplify things. Let's say I want to execute set permissions on the storage directory:

```
[
  {
    "name": "server-1",
    "beforeHooks": "chmod -R 0777 ${RELEASE_PATH}/storage",
    "afterHooks": "",
    "path": "/var/www/html"
  }
]
```

To make these environment variables available, you need to explicitly define which variables you want to pass via the envs input variable.

## Activating the release

Time for the most exciting part, if I say so myself, activating our release. We will re-use a big chunk of our previous step with a couple of changes.
  
  ```
  # //
activate-release:
  name: "${{ matrix.server.name }}: Activate release"
  runs-on: ubuntu-latest
  needs: [ create-deployment-artifacts, prepare-release-on-servers, run-before-hooks ]
  strategy:
    matrix:
      server: ${{ fromJson(needs.create-deployment-artifacts.outputs.DEPLOYMENT_MATRIX) }}
  steps:
    - name: Activate release
      uses: appleboy/ssh-action@master
      env:
        GITHUB_SHA: ${{ github.sha }}
        RELEASE_PATH: ${{ matrix.server.path }}/releases/${{ github.sha }}
        ACTIVE_RELEASE_PATH: ${{ matrix.server.path }}/current
        STORAGE_PATH: ${{ matrix.server.path }}/storage
        BASE_PATH: ${{ matrix.server.path }}
        LARAVEL_ENV: ${{ secrets.LARAVEL_ENV }}
      with:
        host: ${{ matrix.server.ip }}
        username: ${{ matrix.server.username }}
        key: ${{ secrets.SSH_KEY }}
        port: ${{ matrix.server.port }}
        envs: GITHUB_SHA,RELEASE_PATH,ACTIVE_RELEASE_PATH,STORAGE_PATH,BASE_PATH,ENV_PATH,LARAVEL_ENV
        script: |
          printf "%s" "$LARAVEL_ENV" > "${BASE_PATH}/.env"
          ln -s -f ${BASE_PATH}/.env $RELEASE_PATH
          ln -s -f $STORAGE_PATH $RELEASE_PATH
          ln -s -n -f $RELEASE_PATH $ACTIVE_RELEASE_PATH
          service php8.1-fpm reload
  ```
  
  Again, we update the need input variable to include all previous steps before running the activate-release job:

```needs: [ create-deployment-artifacts, prepare-release-on-servers, run-before-hooks ]```

I've added a new Laravel environment variable, LARAVEL_ENV, which will contain the environment variable for our Laravel application. This variable doesn't contain any data just yet. So let's do this first, you can define key-value secrets per repository via the GitHub UI (repository->settings ->secrets)

Let's take a closer look at our bash script line by line:
  
  ```
 # Store the environment data to the /var/www/html/.env file 
printf "%s" "$LARAVEL_ENV" > "${BASE_PATH}/.env"

# Link /var/www/html/.env file to /var/www/html/releases/633be605b03169ef96c2cee1f756852e1ceb2688/.env
ln -s -f ${BASE_PATH}/.env $RELEASE_PATH

# Link /var/www/html/storage directory to /var/www/html/releases/633be605b03169ef96c2cee1f756852e1ceb2688/storage
ln -s -f $STORAGE_PATH $RELEASE_PATH

# Link the release path to the active release path, /var/www/html/current -> /var/www/html/releases/633be605b03169ef96c2cee1f756852e1ceb2688
ln -s -n -f $RELEASE_PATH $ACTIVE_RELEASE_PATH

# Reload php8.1 to detect file changes
service php8.1-fpm reload 
```
  
_Tip: There is a minor delay between the activation and the after-hook job. So make sure to include critical commands in the activation job. You could add an extra release hook configuration if you want to define this per server._

Taking a quick look at all the directories via SSH verifies shows that everything is working correctly.

The current directory is pointing to /var/www/html/releases/5b62b9a13...

The current release environment file **/var/www/html/current/.env** is pointing to **/var/www/html/.env** and **/var/www/html/current/storage** is pointing to **/var/www/html/storage.**

Our Laravel application is up and running with zero downtime! Hurray!

## After hook

Our after-hook is exactly the same as our before-hook besides a few naming changes:
  
 ```
 # //
run-after-hooks:
  name: "${{ matrix.server.name }}: After hook"
  runs-on: ubuntu-latest
  needs: [ create-deployment-artifacts, prepare-release-on-servers, run-before-hooks, activate-release ]
  strategy:
    matrix:
      server: ${{ fromJson(needs.create-deployment-artifacts.outputs.DEPLOYMENT_MATRIX) }}
  steps:
    - name: Run after hooks
      uses: appleboy/ssh-action@master
      env:
        GITHUB_SHA: ${{ github.sha }}
        RELEASE_PATH: ${{ matrix.server.path }}/releases/${{ github.sha }}
        ACTIVE_RELEASE_PATH: ${{ matrix.server.path }}/current
        STORAGE_PATH: ${{ matrix.server.path }}/storage
        BASE_PATH: ${{ matrix.server.path }}
      with:
        host: ${{ matrix.server.ip }}
        username: ${{ matrix.server.username }}
        key: ${{ secrets.SSH_KEY }}
        port: ${{ matrix.server.port }}
        envs: GITHUB_SHA,RELEASE_PATH,ACTIVE_RELEASE_PATH,STORAGE_PATH,BASE_PATH
        script: |
          ${{ matrix.server.afterHooks }}
  ```
  
  Again the needs the input variable is updated to make sure all previous steps have been completed.

## Cleaning up

Every release we upload and extract on our production servers takes up space. You don't want to end up having servers meltdown because of their hard drives being full. So let's clean up after each deployment and keep 5 releases.

```
clean-up:
  name: "${{ matrix.server.name }}: Clean up"
  runs-on: ubuntu-latest
  needs: [ create-deployment-artifacts, prepare-release-on-servers, run-before-hooks, activate-release, run-after-hooks ]
  strategy:
    matrix:
      server: ${{ fromJson(needs.create-deployment-artifacts.outputs.DEPLOYMENT_MATRIX) }}
  steps:
    - name: Run after hooks
      uses: appleboy/ssh-action@master
      env:
        RELEASES_PATH: ${{ matrix.server.path }}/releases
        ARTIFACTS_PATH: ${{ matrix.server.path }}/artifacts
      with:
        host: ${{ matrix.server.ip }}
        username: ${{ matrix.server.username }}
        key: ${{ secrets.SSH_KEY }}
        port: ${{ matrix.server.port }}
        envs: RELEASES_PATH
        script: |
          cd $RELEASES_PATH && ls -t -1 | tail -n +6 | xargs rm -rf
          cd $ARTIFACTS_PATH && ls -t -1 | tail -n +6 | xargs rm -rf
```

Let's take a closer look at the commands we are executing:

```
cd $RELEASES_PATH && ls -t -1 | tail -n +6 | xargs rm -rf
cd $ARTIFACTS_PATH && ls -t -1 | tail -n +6 | xargs rm -rf
```

This will cd into our release and artifacts directory, list all files in a given directory, order the list by timestamp, return the results with the offset of 5 entries, and finally remove given files or folders.

##   Multi-server deployment
To deploy to multiple servers, you only need to update your deployment-config.json to include the additional servers:

```
[   
    {     
        "name": "server-1",    
        "ip": "123.456.78.90",     
        "username": "web",     
        "port": "22",     
        "beforeHooks": "",     
        "afterHooks": "",     
        "path": "/var/www/html"   
      }, 
  
      {    
         "name": "server-2",     
          "ip": "901.234.56.78",     
          "username": "web",     
          "port": "22",     
          "beforeHooks": "",     
          "afterHooks": "",     
          "path": "/var/www/html"  
       }   
]
```

## Conclusion
That's a wrap! You've successfully made a release artifact from your application build and deployed it to multiple servers without any downtime!
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  






