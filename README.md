# Development workflow using _git submodules_ and _docker_

This setup aims at simplifying the development workflow and increasing its
efficiency no matter whether the developer is (remote, together with the dev
team), and the experience she/he has with the system(familiar with the full
system or just contributing to a specific part).

## Table of Content
<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Table of Content](#table-of-content)
- [Credits](#credits)
- [Prerequisites](#prerequisites)
	- [Git configuration](#git-configuration)
- [Background and Assumptions](#background-and-assumptions)
- [Creating the git repo for the development environment](#creating-the-git-repo-for-the-development-environment)
- [Adding submodules](#adding-submodules)
- [Removing a submodule](#removing-a-submodule)
	- [Temporarily removing a submodule](#temporarily-removing-a-submodule)
	- [Permanently removing a submodule](#permanently-removing-a-submodule)
- [Working with container and submodules](#working-with-container-and-submodules)
	- [Setting up the development environment as a developer](#setting-up-the-development-environment-as-a-developer)
	- [Working independently on a module](#working-independently-on-a-module)
	- [Getting an update from the submodule’s remote](#getting-an-update-from-the-submodules-remote)
	- [Making updates to the container](#making-updates-to-the-container)
	- [Grabbing container updates](#grabbing-container-updates)
	- [Updating a submodule in-place in the container](#updating-a-submodule-in-place-in-the-container)
- [Dockerizing everything](#dockerizing-everything)
	- [Quick start](#quick-start)
	- [How it's done](#how-its-done)
		- [arangodb](#arangodb)
		- [server](#server)
		- [web](#web)
		- [nginx - reverse proxy and router](#nginx-reverse-proxy-and-router)
- [Conclusion](#conclusion)

<!-- /TOC -->

## Credits

* The original idea by Amine Mouafik [Efficient development workflow using Git submodules and Docker Compose](https://www.airpair.com/docker/posts/efficiant-development-workfow-using-git-submodules-and-docker-compose).
* The excellent [git documentation](https://git-scm.com/doc)
* [Mastering Git submodules](https://medium.com/@porteneuve/mastering-git-submodules-34c65e940407#.s51gcqy2p)
  by Christophe Porteneuve

## Prerequisites

```shell
$ git -v
$ docker -v
$ docker-compose -v
```
Detailed instructions for installing and configuring the above tools are
available on the internet:
* git: https://git-scm.com/book/en/v2/Getting-Started-Installing-Git
* docker: https://docs.docker.com/engine/installation/
* docker-compose: https://docs.docker.com/compose/install/

Docker containers are an important part of this development stack and its
associated workflow. Docker tools must be properly installed and configured
on the devloper workstation.

> On Mac OS X, it is not recommended to use the Docker Toolbox due to several
> issues with volumes through the virtualbox VM. The newer Docker for Mac,
> which comes with the lightweight [xhyve](https://github.com/mist64/xhyve/)
> virtualization layer, should be used instead.

Git is the chosen SCM for this development environment/workflow and git
submodules are an important part of it. The same principles and methodology
can be used with other SCM tools as long as they can provide a similar
mechanism than git submodules to integrate source for multiple repositories
into the file tree of a master project.

> Git submodules are tricky to use and may seem a bit daunting at first but
> they have been chosen for this workflow over other alternatives for the
> following reasons:
> * built into git with no need for any extra tools
> * flexibility to work with each module as a separate repository or to
>   work from within the enclosing project.
>
> Alternative to git submodules include
[git subtree](https://github.com/git/git/blob/master/contrib/subtree/git-subtree.txt)
and [git-subrepo _(3rd party tool)_](https://github.com/ingydotnet/git-subrepo).

### Git configuration
To make the work with git submodules more reliable, it is strongly recommended
to have the following git configuration setup:
```shell
$ git config --global diff.submodule log
```
Equivalent to `git diff --submodules=log`.

```shell
git config --global status.submoduleSummary true
```
Extend the status information to add the commits in the included submodules.

```shell
git config --global alias.spush 'push --recurse-submodules=on-demand'
```
An alias that can be used when pushing commits to the remote repos to make sure
that all submodules commits are pushed when pushing the main project.
> Failure to do so will create issues when developers need to update their
> environment with missing commits from the submodules and errors when trying
> to update them.

## Background and Assumptions

The examplelication we will be working on is composed of the following modules:
* arango: the backend database, using [ArangoDB](https://www.arangodb.com)
* server: server examplelication, using [node.js](https://nodejs.org/en/), for the
  implementation of the REST API
* web: the public web site for the examplelication, served through
  [nginx](https://www.nginx.com)

The domain name 'example.local' will be used locally for the development environment
and two host entries need to be configured in **/etc/hosts** for the _server_ and
_web_ modules as following:

```
127.0.0.1 api.example.lo
127.0.0.1 www.example.lo
```
To test that the host names are properly setup, simply use ping as following:
```shell
$ ping api.example.lo
PING api.example.lo (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.044 ms

--- api.example.lo ping statistics ---
1 packets transmitted, 1 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.044/0.044/0.044/0.000 ms
```
```shell
$ ping www.example.lo
PING www.example.lo (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.045 ms

--- www.example.lo ping statistics ---
1 packets transmitted, 1 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.045/0.045/0.045/0.000 ms
$
```

We assume that git is running on the localhost as a server, and that the
following repositories already exist:
* /var/git/example/arango.git, for the arango module
* /var/git/example/server.git, for the server module
* /var/git/example/web.git, for the web module

## Creating the git repo for the development environment

> These steps are only required when setting up the development workflow fo the
> first time. Once it's done, developers being onboarded on this workflow only
> need to start with 'Setting up the development environment'

On the git server (localhost), login as the git user and create a new bare
repository with the name 'development-local.git'
```shell
$ git init --bare /var/git/example/development-local.git
Initialized empty Git repository in /var/git/example/development-local.git/
```
We will work on that repo to finish its setup from a regular user on her local
environment.

We'll clone the development-local git repo, add the various application modules
to it, populate it with the docker files we need to configure and manage the
lifecycle of the different docker containers used for the testing.

```shell
$ git clone git@localhost:/var/git/example/development-local.git development-local
Cloning into 'development-local'...
warning: You appear to have cloned an empty repository.
Checking connectivity... done.
$ cd development-local
$ touch README.md
$ git add README.md
$ git commit -m "Empty README.md"
$ git push
```
We have just added an empty README.md file to the development-local repo and
pushed the commit to the remote. The development repo should be populated with
at least the basic workflow documentation. You can use this README.md file and
adapt as necessary for your own needs.

## Adding submodules

Let's add the submodules...
```shell
$ git submodule add git@localhost:/var/git/example/arango.git
$ git submodule add git@localhost:/var/git/example/server.git
$ git submodule add git@localhost:/var/git/example/web.git
```
This will have created a .gitmodules in the development-local repository so
that future developers will be able to fetch the submodules while fetching the
development repo.

By default, submodules will add the subproject into a directory named the same
as the repository, in this case “DbConnector”. You can add a different path at
the end of the command if you want it to go elsewhere.

Finally we can commit the changes and push to the remote (either using the
previously defined alias `git spush` or simply `git push` as we did not make
any changes to the submodules after adding them).

```shell
$ git commit -m "Added submodules"
$ git spush
```

## Removing a submodule
There are two situations where you’d want to “remove” a submodule:
* You just want to clear the working directory (perhaps before archiving the
  container’s WD) but want to retain the possibility of restoring it later (so
  it has to remain in .gitmodules and .git/modules);
* You wish to definitively remove it from the current branch.

Let’s see each case in turn.

### Temporarily removing a submodule
The first situation is easily handled by git submodule deinit.
```shell
$ git submodule deinit web
Cleared directory 'web'
Submodule 'web' (git@localhost:/var/git/example/web.git) unregistered for path 'web'
```
This has no impact on the container status whatsoever. The submodule is not
locally known anymore (it’s gone from .git/config), so its absence from the
working directory goes unnoticed. We still have the vendor/plugins/demo directory
but it’s empty; we could strip it with no consequence.

The submodule must not have any local modifications when you do this, otherwise
you’d need to --force the call.

Any later subcommand of git submodule will blissfully ignore this submodule until
you init it again, as the submodule won’t even be in local config. Such commands
include update, foreach and sync.

On the other hand, the submodule remains defined in .gitmodules: an init followed
by an update (or a single update --init) will restore it as new:

```shell
$ git submodule update --init web
Submodule 'web' (git@localhost:/var/git/example/web.git) registered for path 'web'
Submodule path 'web': checked out 'ab2e7566ad972378dd51e98dff74250a4ff58b28'
```

### Permanently removing a submodule
This means you want to get rid of the submodule for good: a regular git rm will
do, just like for any other part of the working directory. This will only work
if your submodule uses a gitfile (a .git which is a file, not a directory),
which is the case starting with Git 1.7.8.

In addition to stripping the submodule from the working directory, the command
will update the .gitmodules file so it does not reference the submodule anymore.

For a comprehensive cleanup, it is recommended to do both in sequence, first the
`deinit` then the `rm` so as not to end up with the module still in the local
config.

```shell
$ git submodule deinit web
$ git rm web
$ git commit -m "Removed web module"
$ git push
```
Regardless of the approach, the submodule’s repo remains present in
.git/modules/web. If you need to add the module back again, you need to kill
that before.

## Working with container and submodules

### Setting up the development environment as a developer

To get quickly setup as a developer, you simply need to do the following
instructions (after ensuring the prerequisites and additional configuration for
git is done):

```shell
$ git clone --recursive git@localhost:/var/git/example/development-local.git development-local
```
This will have the effect of cloning the development-local repo, initializing
(`git submodule init`) and updating (`git submodule update`) the enclosed
submodules, all in one go.

```
development-local
├── README.md
├── arango
│   └── README.md
├── server
│   └── README.md
└── web
    └── README.md
```

Since Git 1.7.8, submodules use a simple .git file with a single gitdir: line
mentioning a relative path to the actual repo folder, now located inside the
container’s .git/modules. This is mostly useful when the container has branches
that don’t have the submodule at all: this avoid having to scrap the submodule’s
repo when switching to such a container branch.
```shell
$ cd web
$ ls -la
total 8
drwxr-xr-x  4 abdessattar  staff  136 Jul 16 14:47 .
drwxr-xr-x  8 abdessattar  staff  272 Jul 16 14:47 ..
-rw-r--r--  1 abdessattar  staff   84 Jul 16 14:47 .git
-rw-r--r--  1 abdessattar  staff    0 Jul 16 14:47 README.md
$ cat .git
gitdir: /Users/abdessattar/Documents/example-dev/development-local/.git/modules/web
```

Be that as it may, the container and the submodule truly act as independent
repos: they each have their own history (log), status, diff, etc. Therefore be
mindful of your current directory when reading your prompt or typing commands:
depending on whether you’re inside the submodule or outside of it, the context
and impact of your commands differ drastically!

> The container and the submodule truly act as independent repos.

Finally, the submodule commit referenced by the container is stored using its
SHA1, not a volatile reference (such as a branch name). Because of this, a
submodule does not automatically upgrade which is a blessing in disguise when
it comes to reliability, maintenance and QA.

Because of this, most of the time a submodule is in detached head state inside
its containers, as it’s updated by checking out a SHA1 (regardless of whether
that commit is the branch tip at that time).
```shell
$ cd web
$ git status
HEAD detached at ab2e756
nothing to commit, working directory clean
```

### Working independently on a module

It is perfectly fine to work on a module independently of the development setup.
We will see later how we can grab the updated made to a submodule from its
remote and integrate them into the development container.

Let's assume a developer needs to add a node.js basic application scaffolding
into the server submodule. The updated will be done independently on the server
repo.

```shell
$ git clone git@localhost:/var/git/example/server.git server
```

Add an `index.js` file:
```js
// index.js
var express = require('express');
var app = express();

app.get('/hello', function (req, res) {
  res.send('Hello World from node.js!');
});

app.listen(3000, function () {
  console.log('Example app listening on port 3000!');
});
```

Add a `package.json` file:
```json
# package.json
{
  "name": "example",
  "version": "1.0.0",
  "description": "Example application server",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "repository": {
    "type": "git",
    "url": "git@localhost:/var/git/example/server.git"
  },
  "dependencies": {
    "express": "^4.14.0"
  }
}
```

Add `.gitignore` file:
```
node_modules
```

Commit updates and push to remote:
```shell
$ git add -all
$ git commit -m "Added simple hello world node/express app"
$ git push

-> The rsulting history tree would like like this:

(10 seconds ago)    * 03d50da  (HEAD -> master, origin/master, origin/HEAD)
                    |
   (3 hours ago)    * ab2e756
```

### Getting an update from the submodule’s remote

Let's assume now that a second developer is setting up a full development
environment will all submodules.

```shell
$ git clone --recursive git@localhost:/var/git/example/development-local.git development-local

$ cd development-local
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working directory clean

$ git submodule status
3a16594c9272c2c521801bac93d3ac958deca336 arango (heads/master)
ab2e7566ad972378dd51e98dff74250a4ff58b28 server (ab2e756)
ab2e7566ad972378dd51e98dff74250a4ff58b28 web (heads/master)

$ cd server
$ git status
HEAD detached at ab2e756
nothing to commit, working directory clean
```

Notice that the server module is in a detached HEAD mode and no longer tracking
the remote master because it has been modified on the remote and the container
only includes the submodule through a specific commit SHA1 (ab2e756).

Suppose we want to grab the latest commits made on the remote/master in our
local submodule.

> On a side note, I would not recommend using pull for this kind of update.
> To properly get the updates in the working directory, this command requires
> that you’re on the proper active branch, which you usually aren’t (you’re on
> a detached head most of the time). You’d have to start with a checkout of
> that branch. But more importantly, the remote branch could very well have
> moved further ahead since the commit you want to set on, and a pull would
> inject commits you may not want in your local codebase.

Therefore, I recommend splitting the process manually: first git fetch to get
all new data from the remote in local cache, then log to verify what you have
and checkout on the desired SHA1. In addition to finer-grained control, this
approach has the added benefit of working regardless of your current state
(active branch or detached head).

```shell
$ git fetch

$ git log --oneline origin/master
03d50da Added simple hello world node/express app
ab2e756 Empty README.md

$ git checkout 03d50da
Previous HEAD position was ab2e756... Empty README.md
HEAD is now at 03d50da... Added simple hello world node/express app

$ ls -1
README.md
index.js
package.json
```
As a result of the above, the parent container is modified and we can see the
detailed status as below:
```shell
$ cd -
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   server (new commits)

Submodules changed but not updated:

* server ab2e756...03d50da (1):
  > Added simple hello world node/express app

no changes added to commit (use "git add" and/or "git commit -a")
```

We now only need to perform the container commit that finalizes our submodule’s
update.

> If you had to touch the container’s code to make it work with this update,
> commit it along, naturally. On the other hand, avoid mixing submodule-related
> changes and other stuff that would just pertain to the container code: by
> neatly separating the two, later migrations to other code-reuse approaches
> are made easier (also, as usual, atomic commits FTW).

```shell
$ git commit -am "Moved server submodule to 03d50da"
$ git push
```

### Making updates to the container

Making changes and committing/pushing them within the container and not the
submodules is starightforward. It is however recommended to keep such updates
separate from submodule version changes and updates for the sake of better
code management.

Let's update the container with an enhanced REDME.md file that contains detailed
instructions and push the update to the remote.

```shell
$ vim README.md
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   README.md

no changes added to commit (use "git add" and/or "git commit -a")

$ git commit -am "Updated README.md"
$ git push
```

### Grabbing container updates

When pulling updates from the conatiner repo's remote, **Git auto-fetches, but
does not auto-update**. The local cache is up-to-date with the submodule’s
remote, but the submodule’s working directory stays with its former contents.
Although this auto-fetching is limited to already-known submodules: any new
ones, not yet copied into local configuration, are not auto-fetched.

> If you don’t explicitly update the submodule’s working directory, your next
> container commit will regress the submodule. Is is therefore mandatory that
> you finalize the update.

```shell
$ git pull
$ git submodule sync --recursive
$ git submodule update --init --recursive
```

### Updating a submodule in-place in the container

This scenario should be limited to the cases where a submodule cannot be
compiled or tested outside the container. Otherwise it is preferred to modify
submodules within their independent repos and pull the updates in the container
afterwards.

Let's update the web submodule to add a simple index.html page.

First, we need to make sure we are starting from a clean start, which means the
tip of a branch. Let's assume we can add on top of the current submodule master
branch without breaking the rest of the container and its submodules (if the
risk of breaking the container happens often, it probably means that embedding
this code as a submodule was not a good design architectural choice, and it maybe
more appropriate to just embed its code inline in the container or somewhere else).

```shell
$ cd web
$ git checkout master
$ git pull --rebase
```

We can now edit the code, test it etc. then perform the **two commits and the
two necessary pushes** to get the updates to the remote so other developers
can see them them and grab them.
```shell
$ git add index.html
$ git status -s
A  index.html
```

Commit the submodule updates:
```shell
$ git commit -am "Added simple index.html"
```

Commit the container to move the submodule to the new version:
```shell
$ cd ..
$ git commit -am "Moving submodule web to 7ff5ca7"
```

Finally, push the container and the submodule either in one go with
`git push --recurse-submodules=on-demand` or by doing two `push` operations,
one in the submodule and one in the container.
```shell
$ git push --recurse-submodules=on-demand
```

## Dockerizing everything

### Quick start

All modules are dockerized. To get started with the testing, clone the
development repo as described in '[Setting up the development environment as a developer](#setting-up-the-development-environment-as-a-developer)'
and do this from the development repo root:

```shell
$ cd server && npm install
$ docker-compose up
```

To tear down the docker containers, simply do this from the development
repo root:
```shell
$ docker-compose down
```

### How it's done

Docker containers enable us to automate the deployment of applications or
application modules inside lightweight containers. No more spending hours or
days making sure all dependencies with correct versions are downloaded and
installed. No more problems due to environment pollution on the test systems.
Simply pull the right container, deploy it and run it.

With this docker-compose comes to play as a simple but straightforward
tool to define and run multi-container applications, providing basic
orchestration capabilities.

There are certainly many more sophisticated tools and platforms, such as Apache
Mesos, Kubernetes, etc... to manage complex environments and orchestrate large
clusters of multi-container applications. But we are here for the main purpose
of efficiently setting up a development environment and supporting our
multi-module development workflow with the minimal upfront requirements.

As we will see, docker and docker-compose will allow us to build everything we
need and get supported our workflow with simply a docker-compose descriptor
file. That in itself is amazing!

Let's recall the overall application architecture:
* We mainly provide a set of REST APIs served out of our node.js/express
  application server, located in the `server` module.
* Our `server` modules relies on a backend database, implemented using ArangoDB.
  The choice of ArangoDB is motivated by its versatility to support document
  storage and graph databases. Any other alternate database that satisfies the
  need may be used, and a docker container would most likely be available of
  docker hub. The scripts and optional Foxx microservices files for ArangoDB are
  hosted in our `arango` module.
* We also provide a nice web site for our app, served by nginx our of the `web`
  module. The web site being fully static or using dynamic pages would not
  impact at all our deployment architecture. We simply would need to augment
  the container with the required additional capabilities.
* Finally, to protect our application modules and enable us to leverage all the
  goodies that can come with `nginx`, we front all our external interfaces with
  a reverse proxy. There is no special module in our application repo dor that.
  The whole thing can be implemented within the nginx container definition.
  Should we need to add some artificats in the future to support enhancements
  to our nginx reverse proxy, such as certificates and keys for TLS, we simply
  need to create a new application module and add it to our development repo.

The required containers and their resources are summarized in the table below:

| Container | Main function                                  | Baseline Docker Container                                                                              | Volumes                                                                                         | Ports     |
|-----------|------------------------------------------------|--------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|-----------|
| arangodb  | backend document and graph database            | Official container image from docker hub arangodb/arangodb                                             | data: docker volume apps: optional Foxx microservices source code, embedded in arango submodule | 8529:8529 |
| server    | REST API and business logic application server | Official stable release of node.js from docker hub node:argon                                          | app: mapped from the server submodule                                                           | 3000:3000 |
| web       | public web site for the application            | Alpine nginx container image from docker hub evild/alpine-nginx                                        | site: mapped from the web submodule config: mapped from the web submodule                       | 8000:80   |
| nginx     | reverse proxy                                  | The amazing auto-configured reverse proxy container based on nginx from docker hub jwilder/nginx-proxy | internal mapping needed for the docker.sock, not related to our application modules             | 80:80     |

> NOTE: docker-compose.yml uses version 2 configuration format to get access to
> the volume declarations and to get a cleaner overall descriptor.

#### arangodb

A local volume created and managed by docker is used for the database storage,
while we use our arango submodule to provide ArangoDB with the application
source code for Foxx microservices.

```yml
arangodb:
  image: arangodb/arangodb
  environment:
    - ARANGO_NO_AUTH=1
  ports:
    - "8529:8529"
  volumes:
    - arangodb-data:/var/lib/arangodb3/
    - arangodb-apps:/var/lib/arangodb3-apps/
    - ./arango/foxx:/var/lib/example-apps/:ro
    - ./arango/scripts/:/var/lib/arangodb3-scripts/:ro
```

#### server

We need to link this container to the `arangodb` container for our application
server to be able to communicate with the database via the exposed ports.

```yml
server:
  image: node:argon
  volumes:
    - ./server/:/server/
  working_dir: /server/
  command: npm start
  environment:
    - VIRTUAL_HOST=api.example.lo
    - VIRTUAL_PORT=3000
  ports:
    - "3000:3000"
  links:
    - arangodb
```

#### web

We don't want to override the entire nginx configuration filesystem in the
container, so we only mount volumes for the `nginx.conf` file and for the
`conf.d/` sub-folder.

```yml
web:
  image: evild/alpine-nginx
  volumes:
    - ./web/public/:/usr/share/nginx/html/:ro
    - ./web/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    - ./web/nginx/conf.d/:/etc/nginx/conf.d/:ro
  environment:
    - VIRTUAL_HOST=www.example.lo
    - VIRTUAL_PORT=80
  ports:
    - "8000:80"
```

#### nginx - reverse proxy and router

J. Wilder's nginx reverse proxy container provides an easy and straightforward
method to automatically and dynamically configure all our containers for
reverse proxying by nginx. To mark a container for configuration inside nginx,
you simply add the environment variables `VIRTUAL_HOST` and `VIRTUAL_PORT` to
its container definition.

For our application, we have the following needs:
* Everything coming to `app.example.lo` should be served by the node.js/express
  inside the `server` container.
* Everything coming to `www.example.lo` should be served by the nginx web server
  inside the `web` container.

> The virtual hosts are locally defined in the machine's /etc/hosts file for
> the purpose of our development workflow. In production they can be replaced
> with the real hosts/domain.

J. Wilder's automated proxy container can do a lot more and can be extensiely
customized to support complex configurations of nginx, TLS, multiple virtual
hosts, etc... For reference, the full documentation is available at
https://github.com/jwilder/nginx-proxy.

```yml
nginx:
  image: jwilder/nginx-proxy
  volumes:
    - /var/run/docker.sock:/tmp/docker.sock
  ports:
    - "80:80"
```

## Conclusion

With a fully working developer workflow, supported by git and docker, we can
get a full developer team or a single developer started in a matter of minutes
rather than days. Moreover, with the flexibility that is offered by docker
and the power of git submodules, we can easily adapt the application sturctures
to add more modules.

Microservices architectures, coupled with containers and container composition
are powerful software architecture tools to build powerful systems while
significantly reducing the complexity overhead on the development workflows.
