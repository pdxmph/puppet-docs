---
layout: default
title: "Hiera 1: Complete Example"
description: "Learn how to use Hiera to replace a module's conditional code with this walkthrough."
---

[ntp_module]: http://forge.puppetlabs.com/puppetlabs/ntp
[hiera.yaml]: http://docs.puppetlabs.com/hiera/1/configuring.html
[Hiera command line tool]: http://docs.puppetlabs.com/hiera/1/command_line.html

The best way to show how Hiera can simplify your Puppet manifests and help you avoid repetitive code is to take some existing Puppet code and refactor it to work with Hiera.  

In this example, we'll use the popular [Puppet Labs ntp module][ntp_module], an exemplar of the package/file/service pattern in common use among Puppet users, and one that will involve increasing amounts of conditional logic to expand the kinds of machines it can configure. By the time we're through, Hiera will have replaced that conditional logic and we'll have demonstrated further ways Hiera can simplify configuration and reduce the amount of code you have to write to configure your systems.

Since we want to get you to a place where you can try everything out right away, we're providing a directory with all the configuration files and module code. You can view the files [here] or [download them all as a zip file], and we'll link to them as we progress through this walkthrough.

##  What Can We Simplify?

### Remove Conditional Logic

The ntp module is fairly concise: On four supported operating systems, it installs the ntp package, then configures and manages the ntp service. It includes 84 lines of conditional logic to check for the operating system in use on a given node then determine the name of the ntp package and service then select a template file to use for configuration. By removing that conditional logic, we'll be able to reduce the size of the ntp module's `init.pp` manifest by almost half.

### Simplify and Separate Site Configuration Data 

The ntp class provided by the module also requires a number of parameters, each of which would have to be declared each time the class is applied to a given node. That adds a lot of overhead to our site manifests. Hiera will allow us to declare those parameters in the context of our hierarchy of data sources, simplifying the process of changing configuration by isolating that organizational data from the module itself and from our site manifests. 

### Manage Node Classification

We can also use Hiera to assign classes to nodes. This isn't a feature of the ntp module we'll be using, but rather an added capability Hiera provides us, allowing us to add a single line to our `site.pp` manifest, which in turn will allow us to assign classes to nodes from within our Hiera data sources.

## Describing Our Environment

For purposes of this walkthrough, we'll assume a situation that looks something like this:

- We have a number of different servers in use running four different operating systems, some of these are physical servers, some are virtual. **use hiera_include to assign ntp class** 
- In the case of the virtualized servers, we manage their time via their host operating systems, so we don't want them to run an ntp service. **use virtualized fact to disable ntp on these machines**
- Some of these servers should be able to act as ntp servers for other machines in the organization. **use "node" hierarchy to toggle restrict parameter**
- All of these servers should first use organizational ntp services where possible, but be able to fall back to time servers managed by
trusted outside organizations if organizational servers aren't available. **use hiera array to gather both org ntp servers and those provided by distros, etc.** 

## Configuring Hiera and Setting Up the Hierarchy

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

The [Hiera command line tool][] is useful when you're in the process of designing and testing your hierarchy. You can use it to mock in facter facts for Hiera to look up without having to go through cumbersome trial-and-error puppet runs. Since the `hiera` command expects to find `hiera.yaml` in `/etc/hiera.yaml`, you should set a symbolic link from your `hiera.yaml` file to `/etc/hiera.yaml`. 


## Writing the Data Sources

We're not going to show every single data source. As we mentioned at the top, you can download all the files we reference here and start using them on a test system right away.  We will, however, provide an example of each to give you a sense of how to write one. 

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

