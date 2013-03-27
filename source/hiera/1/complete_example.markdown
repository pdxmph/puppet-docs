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

Making these sorts of decisions --- decisions that are specific to an individual organization --- is the main strength of Hiera. You can use Hiera for these sorts of decisions right away, without having to rewrite a single line of your module. Using Hiera data sources to organize these decisions makes it easier to share a module with others: You can keep "organizational truth" in Hiera, independent of your module code. 

### Remove Conditional Logic For Facts We Don't Control

A second thing we can simplify is the conditional logic our ntp module has to include to deal with the many different names for packages, services, and configuration files assorted operating systems use:

The ntp module is fairly concise: On four supported operating systems, it installs the ntp package, then configures and manages the ntp service. There are only 21 lines of code in the manifest that actually describe the needed configuration. But it includes [84 lines of conditional logic][ntp_module_conditional] to check for the operating system in use on a given node, then determine the name of the ntp package and service, *then* identify a template file to use for configuration. By removing that conditional logic, we'll be able to reduce the size of the ntp module's `init.pp` manifest by almost half.

The downside of this kind of simplifcation is that it makes your Hiera data sources more complex and it makes your modules a little harder to share. It's no longer enough to cover the five user-facing parameters the module originally required: You also have to introduce data sources to cover your operating systems. This is a shortcoming of Hiera as it currently exists, and it's [an issue we're working on][hiera_module_data_ticket]. 

We're going to show how to move some of this conditional logic into Hiera, but we recommend it only in cases where you don't plan to share your module with others. 

### Classify Nodes With Hiera

We can also use Hiera to assign classes to nodes. This wasn't a feature of the ntp module we'll be using to begin with, but rather an added capability Hiera provides us, allowing us to use the [hiera_include][] function to add a single line to our `site.pp` manifest, which in turn will allow us to assign classes to nodes from within our Hiera data sources.

## Describing Our Environment

For purposes of this walkthrough, we'll assume a situation that looks something like this:

- We have a number of different servers in use running four different operating systems, some of these are physical servers, some are virtual. **use hiera_include to assign ntp class** 
- In the case of the virtualized servers, we manage their time via their host operating systems, so we don't want them to run an ntp service. **use virtualized fact to disable ntp on these machines**
- Some of these servers should be able to act as ntp servers for other machines in the organization. **use "node" hierarchy to toggle restrict parameter**
- All of these servers should first use organizational ntp services where possible, but be able to fall back to time servers managed by
trusted outside organizations if organizational servers aren't available. **use hiera array to gather both org ntp servers and those provided by distros, etc.** 

## Case 1. Putting Organization Data in Hiera

We're going to break this walkthrough into two cases: In the first, simpler case, we're going to use the ntp module as is and supply data for the `ntp` class parameters with Hiera. In the the second, more complex case, we're going to remove a big chunk of conditional logic the ntp module relies on to determine the names and locations of assorted ntp packages, services, and configuration files.

In either case, we have to start with a `hiera.yaml` file and a hierarchy.

### Configuring Hiera and Setting Up the Hierarchy

All Hiera configuration begins with `hiera.yaml`. You can read a [full discussion of this file][hiera.yaml], including where you should put it depending on the version of Puppet you're using.  Here's the one we'll be using for this exercise:

	---
	:backends:
	  - json
	:json:
	  :datadir: /etc/puppet/hieradata
	:hierarchy:
	  - node/%{::fqdn}
	  - osfamily/%{::osfamily}
	  - common

Step-by-step:

`:backends:` tells Hiera what kind of data sources it should process. In this case, we'll be using JSON files. 

`:json:` configures the JSON data backend, telling Hiera to look in `/etc/puppet/hiera` for JSON data sources.

`:hierarchy:` configures the data sources Hiera should consult. Puppet users commonly separate their hierarchies into directories to make it easier to get a quick top-level sense of how the hierarchy is put together. In this case, we're keeping it simple: 

Files in the `/node` directory should be named for the fully qualified domain name (`fqdn`) fact for any systems we want to specifically configure with Hiera, and should end with `.json.` Given the nodes `master.acme.com`, `bugs.acme.com`, and `daffy.acme.com`, we'd want to include the following files:

- `master.acme.com.json`
- `bugs.acme.com.json`
- `daffy.acme.com.json`

Files in the `/osfamily` directory should be named for the `osfamily` fact. In this case, we're supporting the Debian, FreeBSD, Red Hat, SUSE, and Arch Linux operating system families. We're going to use the `osfamily` fact to determine the name of the ntp package, the location of its configuration files, the template to use for its configuration files, and the names of appropriate outside ntp servers to use. So we'll need to include the following files in our `osfamily` directory:

- `Archlinux.json`
- `Debian.json`
- `FreeBSD.json`
- `RedHat.json`
- `Suse.json`

Note that these files are all case sensitive and must match the output of the facter command.

Finally, the `common.json` file is a data source that provides any common or default values we want to use when Hiera can't find a match for a given key elsewhere in our hierarchy. In this case, we're going to use it to set common ntp servers and default configuration options for the ntp module. 

### Configuring for the Command Line

The [Hiera command line tool][] is useful when you're in the process of designing and testing your hierarchy. You can use it to mock in facts for Hiera to look up without having to go through cumbersome trial-and-error puppet runs. Since the `hiera` command expects to find `hiera.yaml` at `/etc/hiera.yaml`, you should set a symbolic link from your `hiera.yaml` file to `/etc/hiera.yaml`.

## Writing the Data Sources

Now that we've got Hiera configured, we're ready to return to the ntp module and start figuring out how we can take its conditional logic and convert that into a Hiera hierarchy of data source files. As we mentioned at the top, you can [download all the files][examples] we reference here and start using them on a test system right away. 

> **Learning About Hiera Data Sources:** This example won't cover all the data types you might want to use, and we're only using one of two built-in data backends (JSON). For a more complete look at data sources, please see our guide to [writing Hiera data sources][hiera_datasources], which includes more complete examples written in JSON and YAML. 

### Identifying Parameters

We need to start by figuring out the parameters required by the `ntp` class our module provides. So let's start by looking at the [ntp module's README][], where we see five:

- servers
- restrict
- autoupdate
- enable
- template

**call out osfamily-related logic in ntp's init.pp**

- enumerate the parameters we need to cover
- explain how we're going to break them out across nodes and osfamilies

	
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


### `daffy.acme.com.json`


### `common.json`



> **JSON or YAML?** Out of the box, Hiera supports both YAML and JSON files as data sources. Both work fine, so choosing one is a question of personal preference. We went with JSON, but that doesn't constitute a recommendation. We do recommend you stick to one or the other for simplicity's sake.

## Testing Our Hierarchy

Simple command line test cases:

- simple lookup
- hiera array (to show compilation of all ntp servers)

## Adding a Class to a Node With Hiera

- hiera include

## 

