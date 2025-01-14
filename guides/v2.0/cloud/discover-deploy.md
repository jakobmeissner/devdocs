---
layout: default
group: cloud
subgroup: 01_welcome
title: Deployment process
menu_title: Deployment process
menu_order: 4
menu_node: 
version: 2.0
github_link: cloud/discover-deploy.md
---

#### Contents
*	[Deployment process](#cloud-deploy-over)
*	[GitHub and Bitbucket](#cloud-deploy-over-gh)
*	[Project configuration](#cloud-deploy-conf)
*	[Five phases of deployment](#cloud-deploy-over-phases)
*	[Post-deployment: configure routing](#cloud-deploy-over-phases-route)

## Deployment process {#cloud-deploy-over}
Deploying Magento means simply pushing the source code to your Git repository. The Git repository is part of your projects cluster so it is totally isolated from
other clients.

The built-in Git repository is at the same time "just a normal Git repository" and a very smart piece of software. When you push to it, it will parse the configuration files you committed to your repository so it knows what it needs to deploy.

If you are pushing directly to the Magento Enterprise Cloud Edition Git repository, you will see in your terminal
what is happening in real-time. The same information is going to get streamed in real-time to the Web Interface.

## GitHub and Bitbucket {#cloud-deploy-over-gh}
If you are using external Bitbucket or GitHub repositories, the log
of the operations does not display in the GitHub session. You can still follow what's happening in their interface and in the Magento Enterprise Cloud Edition's Web Interface.

## Project configuration {#cloud-deploy-conf}
What makes it all work is a set of YAML configuration files located in the project root directory. These files define your Magento installation and describe its dependencies. Configuration files specify, for
example, that Magento uses MySQL, some PHP extensions, and Elasticsearch. (These are referred to as *services*.)

## Five phases of deployment {#cloud-deploy-over-phases}
Deployment consists of the following phases:

1.	[Phase 1: Configuration validation and code retrieval](#cloud-deploy-over-phases-conf)
2.	[Phase 2: Build](#cloud-deploy-over-phases-build)
3.	[Phase 3: Prepare slug](#cloud-deploy-over-phases-slug)
4.	[Phase 4: Deploy slugs and cluster](#cloud-deploy-over-phases-slugclus) 
5.	[Phase 5: Deployment hooks](#cloud-deploy-over-phases-hook)
6.	[Post-deployment: configure routing](#cloud-deploy-over-phases-route)

### Phase 1: Configuration validation and code retrieval {#cloud-deploy-over-phases-conf}
The remote server gets your code using Git. The built-in Git server checks the following:

*	Inspects what you sent it. 

	If, for example, you have a syntax error in a configuration file, our Git server refuses the push.
*	Looks for critical vulnerabilities.

	Refusing to push because the built-in Git server detects a vulnerability is a good thing because it means you 
can't break your production system that easily. 

The built-in Git server diffs not only the code but also the
infrastructure (by looking at changes to the `.yaml` configuration files like `.magento.app.yaml`).

Suppose you had a single MySQL database in your cluster and now you want two of those, or maybe you want to add an Elasticsearch instance. The built-in Git server detects this and verifies that the topology of your cluster is modified to your new needs.

This phase also runs `composer install` to retrieve dependencies.

### Phase 2: Build {#cloud-deploy-over-phases-build}
We build only what has changed since the last build. This is one of the things that
make Magento Enterprise Cloud Edition so fast in deployment. 

Magento Enterprise Cloud Edition builds containers, in parallel, each server where something changed. If for example you pushed a
PHP `composer.json` file, it will know it needs to go search for the
dependencies, and shuffle things around to create the correct directory
structure. If in your `.magento.app.yaml` file you also specified a
build hook, you can now run whatever scripts you wish.

It is important to note that at this point the cluster has not been
created yet. So you should not try to connect to a database or imagine
anything was daemonized.

But also know that once the application has been built it is going to be
mounted on a read-only file system (you will be able to configure specific
mount points that are going to be read/write). 

This means you cannot FTP to the server and add modules. Instead, you must add code to your environment and push the environment, which builds and deploys it.

The default Magento build hook is a CLI command called `magento-cloud:build`. It does the following:

*	Enables all extensions
*	Regenerates code and the dependency injection configuration (that is, the Magento `var/generation` and `var/di` directories) using `bin/magento setup:di:compile`.

### Phase 3: Prepare the slug {#cloud-deploy-over-phases-slug}
The result of the build phase is a read-only file system we refer to as a *slug*. In this phase, we create an archive and put it in permanent storage. The next time
you push code, if a service did not change, you can use a slug from the archive.

It also means that reverting a deployment is basically
instantaneous. 

### Phase 4: Deploy slugs and cluster {#cloud-deploy-over-phases-slugclus}
Now we provision your applications and all the backend services you
need:

*	Mounts each service in its own container
*	Mounts the read-write file system is mounted on a highly available distributed storage grid
*	Configures the network so Magento's services can "see" each other (and only each other)

<div class="bs-callout bs-callout-info" id="info">
  <p>The main file system is <em>read-only</em>. This
is what guarantees we can do deterministic deployments. The read-only file system also dramatically improves your site's security because no process can write to the file system.</p>
</div>

### Phase 5: Deployment hooks {#cloud-deploy-over-phases-hook}
The last step runs a deployment script. You can use this for example to anonymize data in development environments, clear caches, ping external continuous integration tools, and so on.

When this script runs, you have access to all the services in your environment (Redis, database, and so on).

The default Magento deployment hook is a CLI command that does the following:

*	If Magento is not installed, it installs Magento with `bin/magento setup:install`, updates the deployment configuration, `app/etc/env.php`, and the database for your specified environment (for example, Redis and website URLs).

*	If Magento is installed, performs any necessary upgrades.

	The deployment script runs [`bin/magento setup:upgrade`]({{ page.baseurl }}install-gde/install/cli/install-cli-subcommands-db-upgr.html) to update the database schema and data (which is necessary after extension or core code updates), and also updates the [deployment configuration]({{ page.baseurl }}config-guide/config/config-php.html), `app/etc/env.php`, and the database for your environment. 

	Finally, the deployment script and clears the Magento cache.

*	Sets the mode to either [`developer`]({{ page.baseurl}}config-guide/bootstrap/magento-modes.html#mode-developer}}) or [`production`]({{ page.baseurl}}config-guide/bootstrap/magento-modes.html#mode-production) based on the environment variable [`APPLICATION_MODE`]({{ page.baseurl }}cloud/env/environment-vars_magento.html).

	In `production` mode, the script generates static web content using the command
	[`magento setup:static-content:deploy`]({{ page.baseurl }}config-guide/cli/config-cli-subcommands-static-view.html).

### Post-deployment: configure routing {#cloud-deploy-over-phases-route}
While the deployment is running, we freeze the incoming traffic at the entry point
so no transactions are lost.

If everything went fine, we are now ready to configure routing so your
web traffic will arrive at your newly created cluster. If something
failed, well then nothing would have happened; the "old cluster" is
still there, so from your users' perspective, nothing changed. 

Neither
failed nor successful deployments result in application downtime.
Because we also route SSH, you can also simply SSH to your cluster, where
you have the same permissions to execute commands as the web server.

#### Related topics
*	[Get started with a project]({{page.baseurl}}cloud/project/project-start.html)
*	[Get started with an environment]({{page.baseurl}}cloud/env/environments-start.html)
*	[`.magento.app.yaml`]({{page.baseurl}}cloud/project/project-conf-files_magento-app.html)
*	[`routes.yaml`]({{page.baseurl}}cloud/project/project-conf-files_routes.html)
*	[`services.yaml`]({{page.baseurl}}cloud/project/project-conf-files_services.html)
