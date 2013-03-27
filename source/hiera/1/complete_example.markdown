---
layout: default
title: "Hiera 1: Complete Example"
description: "Learn how to use Hiera to replace a module's conditional code with this walkthrough."
---

[ntp module's README]: https://github.com/puppetlabs/puppetlabs-ntp/blob/master/README.markdown
[ntp_module]: http://forge.puppetlabs.com/puppetlabs/ntp
[hiera.yaml]: http://docs.puppetlabs.com/hiera/1/configuring.html
[Hiera command line tool]: http://docs.puppetlabs.com/hiera/1/command_line.html
[ntp_init.pp]: https://github.com/puppetlabs/puppetlabs-ntp/blob/master/manifests/init.pp
[template directory]: https://github.com/puppetlabs/puppetlabs-ntp/blob/master/templates
[hiera_datasources]: http://docs.puppetlabs.com/hiera/1/data_sources.html
[examples]: files/hiera_examples/
[ntp_module_conditional]: https://github.com/puppetlabs/puppetlabs-ntp/blob/master/manifests/init.pp#L68-L147
[hiera_include]: http://docs.puppetlabs.com/hiera/1/puppet.html#assigning-classes-to-nodes-with-hiera-hierainclude
[hiera_module_data_ticket]: http://projects.puppetlabs.com/issues/16856
[automatic parameter lookup]: http://docs.puppetlabs.com/hiera/1/puppet.html#automatic-parameter-lookup

The best way to show how Hiera can simplify your Puppet manifests and help you avoid repetitive code is to take some existing Puppet code and refactor it to work with Hiera.  

In this example, we'll use the popular [Puppet Labs ntp module][ntp_module], an exemplar of the package/file/service pattern in common use among Puppet users, and one that will involve increasing amounts of conditional logic to expand the kinds of machines it can configure. By the time we're through, Hiera will have replaced that conditional logic and we'll have demonstrated further ways Hiera can simplify configuration and reduce the amount of code you have to write to configure your systems.

Since we want to get you to a place where you can try everything out right away, we're providing a directory with all the configuration files and module code. You can [view the files here][examples] and download them all as a zip file, and we'll link to them as we progress through this walkthrough.

##  What Can We Simplify?

Let's get started by looking at the ntp module as it currently stands. It does all of its work in a single file, [the `init.pp` manifest][ntp_init.pp], with some ERB templates stored in its [template directory][]. So, what can we simplify with Hiera?

### Express Organizational Information With Hiera

The ntp module takes five parameters:

- servers
- restrict
- autoupdate
- enable
- template

Most of these parameters reflect decisions we have to make about each of the nodes to which we'd apply the ntp module's `ntp` class: Can it act as a time server for other hosts? (`restrict`), which servers should it consult? (`servers`), or  should we allow Puppet to automatically update the ntp package or not? (`autoupdate`). 

Without Hiera, these all represent decisions we would make using conditional logic in our site and node manifests. With Hiera, we can move these decisions into a hierarchy built around the facts that drive these decisions: We can use the fully qualified domain name (`fqdn`) fact to identify the hosts we want to act as ntp servers for our organization, restricting how willing we are to let Puppet update their packages since they're key to our infrastructure. We can use whether or not they're virtual machines (the `is_virtual` fact) to enable or disable the ntp service if we happen to let guest operating systems get their time settings from their host systems.

Making these sorts of decisions --- decisions that are specific to an individual organization --- then expressing them in a hierarchy is the main strength of Hiera. You can use Hiera for these sorts of decisions right away, without having to rewrite a single line of your module. Also, using Hiera data sources to organize these decisions makes it easier to share a module with others: You can keep "organizational truth" in Hiera, independent of your module code. 

### Remove Conditional Logic For Facts We Don't Control

A second thing we can simplify is the conditional logic our ntp module has to include to deal with the many different names for packages, services, and configuration files assorted operating systems use:

The ntp module has fairly modest goals: On four supported operating systems, it installs the ntp package, then configures and manages the ntp service. There are only 21 lines of code in the manifest that actually describe the needed configuration. But it includes [84 lines of conditional logic][ntp_module_conditional] to check for the operating system in use on a given node, identify the name of the ntp package and service, and identify a template file to use for configuration. By removing that conditional logic, we'll be able to reduce the size of the ntp module's `init.pp` manifest by almost half. 

The downside of this kind of simplifcation is that it makes your Hiera data sources more complex and it makes your modules a little harder to share. It's no longer enough to cover the five user-facing parameters the module originally required: You also have to introduce data sources to cover your operating systems, and you'll probably end up scattering module-related data throughout your hierarchy. This is a shortcoming of Hiera as it currently exists, and it's [an issue we're working on][hiera_module_data_ticket]. 

We're still going to show how to move some of this conditional logic into Hiera, but we recommend it only in cases where you don't plan to share your module with others, and where you have a good plan for keeping up with the several places configuration data related to a given module may exist.

### Classify Nodes With Hiera

We can also use Hiera to assign classes to nodes using the [hiera_include][] function, adding a single line to our `site.pp` manifest, then naming assigned nodes in our data sources.

## Describing Our Environment

For purposes of this walkthrough, we'll assume a situation that looks something like this:

- We have two ntp servers in the organization that are allowed to talk to outside time servers. Other ntp servers get their time data from these two servers.
- One of our primary ntp servers is very cautiously configured to keep it from breaking by automatically updating its ntp server package without hands-on testing, but the other is more permissively configured. 
- We have a number of other ntp servers that will use our two primary servers. 

## Case 1. Putting Organization Data in Hiera

We're going to break this walkthrough into two cases: In the first, simpler case, we're going to use the ntp module as is and supply data for the `ntp` class parameters with Hiera. In the the second, more complex case, we're going to remove a big chunk of conditional logic the ntp module relies on to determine the names and locations of assorted ntp packages, services, and configuration files.

In either case, we have to start with a `hiera.yaml` file and a hierarchy.

### Configuring Hiera and Setting Up the Hierarchy

All Hiera configuration begins with `hiera.yaml`. You can read a [full discussion of this file][hiera.yaml], including where you should put it depending on the version of Puppet you're using.  Here's the one we'll be using for this walkthrough:

	---
	:backends:
	  - json
	:json:
	  :datadir: /etc/puppet/hieradata
	:hierarchy:
	  - node/%{::fqdn}
	  - common

Step-by-step:

`:backends:` tells Hiera what kind of data sources it should process. In this case, we'll be using JSON files. 

`:json:` configures the JSON data backend, telling Hiera to look in `/etc/puppet/hiera` for JSON data sources.

`:hierarchy:` configures the data sources Hiera should consult. Puppet users commonly separate their hierarchies into directories to make it easier to get a quick top-level sense of how the hierarchy is put together. In this case, we're keeping it simple: 

We're going to name files in the `/node` directory for the fully qualified domain name (`fqdn`) fact for any nodes we want to specifically configure with Hiera.

Given the nodes  `kermit.acme.com`, and `grover.acme.com`, we'd want to include the following files:

- `kermit.acme.com.json`
- `grover.acme.com.json`

Next, the `common.json` file is a data source that provides any common or default values we want to use when Hiera can't find a match for a given key elsewhere in our hierarchy. In this case, we're going to use it to set common ntp servers and default configuration options for the ntp module. 

### Configuring for the Command Line

The [Hiera command line tool][] is useful when you're in the process of designing and testing your hierarchy. You can use it to mock in facts for Hiera to look up without having to go through cumbersome trial-and-error puppet runs. Since the `hiera` command expects to find `hiera.yaml` at `/etc/hiera.yaml`, you should set a symbolic link from your `hiera.yaml` file to `/etc/hiera.yaml`.

## Writing the Data Sources

Now that we've got Hiera configured, we're ready to return to the ntp module and take a look at the parameters the `ntp` class it provides requires. 

> **Learning About Hiera Data Sources:** This example won't cover all the data types you might want to use, and we're only using one of two built-in data backends (JSON). For a more complete look at data sources, please see our guide to [writing Hiera data sources][hiera_datasources], which includes more complete examples written in JSON and YAML. 

### Identifying Parameters

We need to start by figuring out the parameters required by the `ntp` class our module provides. So let's start by looking at the [ntp module's `init.pp` manifest][ntp_init.pp], where we see five:

servers
: An array of time servers; `UNSET` by default. Conditional logic in `init.pp` provides a list of ntp servers maintained by the respective maintainers of our module's supported operating systems.

restrict
: Whether to restrict ntp daemons from allowing others to use as a server; `true` by default

autoupdate
: Whether to update the ntp package automatically or not; `false` by default

enable
: Automatically start ntp deamon on boot; `true` by default

template
: The name of the template to use to configure the ntp service. This is `undef` by default, and it's configured within the `init.pp` manifest. 

### Making Decisions and Expressing Them in Hiera

Now that we know the parameters the `ntp` class expects, we can start making decisions about the nodes on our system, then expressing those decisions as Hiera data. Let's start with bugs and daffy: The two nodes in our organization that we allow to talk to the outside world for purposes of timekeeping.

#### `kermit.acme.com.json`

We want one of these two nodes, `kermit.acme.com`, to act as the primary organizational time server. We want it to consult outside time servers, we won't want it to update its ntp server package by default, and we definitely want it to launch the ntp service at boot. So let's write that out in JSON, making sure to express our variables as part of the `ntp` namespace to insure Hiera will pick them up as part of its [automatic parameter lookup][].

	{  
	   "ntp::restrict" : false,
	   "ntp::autoupdate" : false,
	   "ntp::enable" : true,
	   "ntp::servers" : [
		   "0.us.pool.ntp.org iburst",
		   "1.us.pool.ntp.org iburst",
		   "2.us.pool.ntp.org iburst",
		   "3.us.pool.ntp.org iburst"
		   ]
	}

Since we want to provide this data for a specific node, and since we're using the `fqdn` fact to identify unique nodes in our hierarchy, we need to save this code in the `/etc/puppet/hiera/node` directory as `kermit.acme.json.com`. 

Once you've saved that, let's do a quick test using the [Hiera command line tool][]:

	$ hiera ntp::servers fqdn=kermit.acme.com

You should see this:

	["0.us.pool.ntp.org iburst", "1.us.pool.ntp.org iburst", "2.us.pool.ntp.org iburst", "3.us.pool.ntp.org iburst"]

That's just the array of outside ntp servers and options we expressed as a JSON array coming back as a puppet array our module will use when it generates configuration files from its templates.


> **Something Went Wrong?** If, instead, you get `nil`, `false`, or something else completely, you should step back through your Hiera configuration making sure:
> - Your `hiera.yaml` file matches the example we provided
> - You've put a symlink to `hiera.yaml` where the command line tool expects to find it (`/etc/hiera.yaml`)
> - You've saved your `kermit.acme.com` data source file with a `.json` extension
> - Your data source file's JSON is well formed. Missing or extraneous commas will cause the JSON parser to fail 

Provided everything works and you get back that array of ntp servers, you're ready to configure another node. 

### `grover.acme.com.json`

Our next ntp node, `grover.acme.com`, is a little less critical to our infrastructure than `bugs`, so we can be a little more permissive with its configuration: It's o.k. if grover's ntp packages are automatically updated. We also want grover to use kermit as its ntp server of choice. Let's express that as JSON:

	{  
	   "ntp::restrict" : false,
	   "ntp::autoupdate" : true,
	   "ntp::enable" : true,
	   "ntp::servers" : [
		   "kermit.acme.com iburst",
		   "0.us.pool.ntp.org iburst",
		   "1.us.pool.ntp.org iburst",
		   "2.us.pool.ntp.org iburst",
		   ]
	}

As with `kermit.acme.com`, we want to save grover's Hiera data source in the `/etc/puppet/hiera/nodes` directory using the `fqdn` fact for the file name: `grover.acme.com.json`. We can once again test it with the hiera command line tool:

	$ hiera ntp::servers fqdn=grover.acme.com
	["kermit.acme.com iburst", "0.us.pool.ntp.org iburst", "1.us.pool.ntp.org iburst", "2.us.pool.ntp.org iburst"]

### `common.json`

So, now we've configured the two nodes in our organization that we'll allow to update from outside ntp servers. However, we still have a few nodes to account for that also provide ntp services. They depend on bugs and daffy to get the correct time, and we don't mind if they update themselves. Let's write that out in JSON:

	{  
	   "ntp::restrict" : false,
	   "ntp::autoupdate" : true,
	   "ntp::enable" : true,
	   "ntp::servers" : [
		   "kermit.acme.com iburst",
		   "grover.acme.com iburst"
		  ]
	}

Unlike kermit and grover, for which we had slightly different but node-specific configuration needs, we're comfortable letting any other node that uses the ntp class use this generic configuration data. Rather than creating a node-specific data source for every possible node on our network that might be running an ntp server, we'll store this data in `/etc/puppet/hiera/common.json`. With our very simple hierarchy, which so far only looks for the `fqdn` fact, any node that doesn't match the nodes we have data sources for will get the data found in `common.json`. Let's test it:

	$ hiera ntp::servers fqdn=snuffie.acme.com
	["kermit.acme.com iburst", "grover.acme.com iburst"]

## Adding a Class to a Node With Hiera

- hiera include

## 


## Case 2. Universal Truth


Modify the hierarchy to include osfamily:

Files in the `/osfamily` directory should be named for the `osfamily` fact. In this case, we're supporting the Debian, FreeBSD, Red Hat, SUSE, and Arch Linux operating system families. We're going to use the `osfamily` fact to determine the name of the ntp package, the location of its configuration files, the template to use for its configuration files, and the names of appropriate outside ntp servers to use. So we'll need to include the following files in our `osfamily` directory:

- `Archlinux.json`
- `Debian.json`
- `FreeBSD.json`
- `RedHat.json`
- `Suse.json`

Note that these files are all case sensitive and must match the output of the facter command.

add to hierarchy: 

- osfamily/%{::osfamily}



	
### `osfamily/%{::osfamily}.json`

**Remove the stuff that's organizational, keep the stuff that's pertinent to the distro**

	{  
	   "ntp::pkg_name" : "ntp",
	   "ntp::svc_name" : "ntp",
	   "ntp::config"   : "/etc/ntp.conf",
	   "ntp::config_tpl" : "ntp.conf.debian.erb",
	   "ntp::supported" : true,
	   "ntp::restrict" : true,
	   "ntp::autoupdate" : true,
	   "ntp::enable" : true,
	   "ntp::ensure" : "running",
	   "ntp::package_ensure" : "latest",
	   "ntp::servers" : [
		   "0.debian.pool.ntp.org iburst",
		   "1.debian.pool.ntp.org iburst",
		   "2.debian.pool.ntp.org iburst",
		   "3.debian.pool.ntp.org iburst"
		   ]
	}





> **JSON or YAML?** Out of the box, Hiera supports both YAML and JSON files as data sources. Both work fine, so choosing one is a question of personal preference. We went with JSON, but that doesn't constitute a recommendation. We do recommend you stick to one or the other for simplicity's sake.
