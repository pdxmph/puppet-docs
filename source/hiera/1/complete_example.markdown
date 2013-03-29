---
layout: default
title: "Hiera 1: Complete Example"
description: "Learn how to use Hiera to replace a module's conditional code with this walkthrough."
---

[external node classifier (ENC)]: http://docs.puppetlabs.com/guides/external_nodes.html
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

In this example, we'll use the popular [Puppet Labs ntp module][ntp_module], an exemplar of the package/file/service pattern in common use among Puppet users, and one that could involve increasing amounts of conditional logic to expand the kinds of machines it can configure. 

We'll start simply, using Hiera to provide the ntp module with parameter data based on particular nodes in our organization. Then we'll use Hiera to assign the `ntp` class provided by the module to specific nodes. Finally, we'll show how Hiera could be used to perform a much more ambitious refactoring of the ntp module to replace a large chunk of conditional logic.

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

Without Hiera, we might find ourselves adding organizational data to our module code as default parameter values, reducing how shareable it is. We might find ourselves repeating configuration data in our site manifests to cover minor differences in configuration between nodes. We might also find ourselves shying away from allowing junior administrators to change much without close supervision, because they have to touch files where a simple mistake could affect our entire Puppet installation.

With Hiera, we can move these decisions into a hierarchy built around the facts that drive these decisions, increase sharability, and repeat ourselves less.

### Classify Nodes With Hiera

We can also use Hiera to assign classes to nodes using the [hiera_include][] function, adding a single line to our `site.pp` manifest, then naming assigned nodes in our data sources.

## Describing Our Environment

For purposes of this walkthrough, we'll assume a situation that looks something like this:

- We have two ntp servers in the organization that are allowed to talk to outside time servers. Other ntp servers get their time data from these two servers.
- One of our primary ntp servers is very cautiously configured to keep it from breaking by automatically updating its ntp server package without testing, but the other is more permissively configured. 
- We have a number of other ntp servers that will use our two primary servers. 

### Our Environment Before Hiera

How did things look before we decided to use Hiera? Classes are assigned to nodes via the puppet sites manifest (`/etc/puppet/manifests/sites.pp` for Puppet Open Source), so here's how our site manifest might have looked:

	node "kermit.vagrant.lan" {
	  class { "ntp":
		servers    => [ '0.us.pool.ntp.org iburst','1.us.pool.ntp.org iburst','2.us.pool.ntp.org iburst','3.us.pool.ntp.org iburst'],
		autoupdate => false,
		restrict => false,
		enable => true,
	  }
	}

	node "grover.vagrant.lan" {
	  class { "ntp":
		servers    => [ 'kermit.acme.com','0.us.pool.ntp.org iburst','1.us.pool.ntp.org iburst','2.us.pool.ntp.org iburst'],
		autoupdate => true,
		restrict => false,
		enable => true,
	  }
	}

	node "snuffie.vagrant.lan", "bigbird.vagrant.lan", "hooper.vagrant.lan" {
	  class { "ntp":
		servers    => [ 'grover.acme.com', 'kermit.acme.com'], 
		autoupdate => true,
		restrict => true,
		enable => true,
	  }
	}

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

Next, the `common.json` file is a data source that provides any common or default values we want to use when Hiera can't find a match for a given key elsewhere in our hierarchy. In this case, we're going to use it to set common ntp servers and default configuration options for the ntp module. 

> **Note:** If you modify `hiera.yaml` between agent runs, you'll have to restart your puppet master for your changes to take effect the next time your puppet agents attempt a run.

### Configuring for the Command Line

The [Hiera command line tool][] is useful when you're in the process of designing and testing your hierarchy. You can use it to mock in
facts for Hiera to look up without having to go through cumbersome trial-and-error puppet runs. Since the `hiera` command expects to find
`hiera.yaml` at `/etc/hiera.yaml`, you should set a symbolic link from your `hiera.yaml` file to `/etc/hiera.yaml`:

	$ ln -s /etc/puppet/hiera.yaml /etc/hiera.yaml

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
: Whether to start the ntp daemon on boot; `true` by default

template
: The name of the template to use to configure the ntp service. This is `undef` by default, and it's configured within the `init.pp` manifest with some conditional logic.

### Making Decisions and Expressing Them in Hiera

Now that we know the parameters the `ntp` class expects, we can start making decisions about the nodes on our system, then expressing those decisions as Hiera data. Let's start with kermit and grover: The two nodes in our organization that we allow to talk to the outside world for purposes of timekeeping.

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
> 
> - Your `hiera.yaml` file matches the example we provided
> - You've put a symlink to `hiera.yaml` where the command line tool expects to find it (`/etc/hiera.yaml`)
> - You've saved your `kermit.acme.com` data source file with a `.json` extension
> - Your data source file's JSON is well formed. Missing or extraneous commas will cause the JSON parser to fail 
> - You restarted your puppet master if you modified `hiera.yaml`

Provided everything works and you get back that array of ntp servers, you're ready to configure another node. 

### `grover.acme.com.json`

Our next ntp node, `grover.acme.com`, is a little less critical to our infrastructure than kermit, so we can be a little more permissive with its configuration: It's o.k. if grover's ntp packages are automatically updated. We also want grover to use kermit as its primary ntp server. Let's express that as JSON:

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

So, now we've configured the two nodes in our organization that we'll allow to update from outside ntp servers. However, we still have a few nodes to account for that also provide ntp services. They depend on kermit and grover to get the correct time, and we don't mind if they update themselves. Let's write that out in JSON:

	{  
	   "ntp::restrict" : true,
	   "ntp::autoupdate" : true,
	   "ntp::enable" : true,
	   "ntp::servers" : [
		   "grover.acme.com iburst",
		   "kermit.acme.com iburst"
		  ]
	}

Unlike kermit and grover, for which we had slightly different but node-specific configuration needs, we're comfortable letting any other node that uses the ntp class use this generic configuration data. Rather than creating a node-specific data source for every possible node on our network that might need to use the ntp module, we'll store this data in `/etc/puppet/hiera/common.json`. With our very simple hierarchy, which so far only looks for the `fqdn` facts, any node that doesn't match the nodes we have data sources for will get the data found in `common.json`. Let's test against one of those nodes:

	$ hiera ntp::servers fqdn=snuffie.acme.com
	["kermit.acme.com iburst", "grover.acme.com iburst"]

### Modifying Our `site.pp` Manifest

Now that everything has tested out from the command line, it's time to get a little benefit from this work in production. 

If you'll remember back to our pre-Hiera configuration, we were declaring a number of parameters for the `ntp` class in our `site.pp` manifest, like this:

	node "kermit.vagrant.lan" {
	  class { "ntp":
		servers    => [ '0.us.pool.ntp.org iburst','1.us.pool.ntp.org iburst','2.us.pool.ntp.org iburst','3.us.pool.ntp.org iburst'],
		autoupdate => false,
		restrict => false,
		enable => true,
	  }
	}

In fact, we had three separate stanzas of that length. But now that we've moved all of that parameter data into Hiera, we can significantly pare down `site.pp`:

	node "kermit.vagrant.lan", "grover.vagrant.lan", "snuffie.vagrant.lan" {
		class { "ntp": }
	}

That's it. 

Since Hiera is providing the parameter data from the data sources in its hierarchy, we don't need to do anything besides assign the `ntp` class to the nodes and let Hiera's parameter lookups do the rest. In the future, as we change or add nodes that need to use the `ntp` class, we can quickly copy data source files to cover cases where a node needs a specialized configuration, or simply add new  nodes that can work with the generic configuration in `common.json` to the list in our easier-to-read `site.pp`.

If you're interested in taking things a step further, using the decision-making skills you picked up in this example to make decisions about which nodes even get a particular class, let's keep going.

## Assigning a Class to a Node With Hiera

In the first part of our example, we were concerned with how to use Hiera to provide data to a parameterized class, but we were assigning the classes to nodes in the traditional Puppet way: By making `class` declarations for each node in our `site.pp` manifest. Thanks to the `hiera_include` function, you can assign nodes to a class the same way you can assign values to class parameters: Picking a facter fact on which you want to base a decision and adding data sources to your `hiera.yaml` file. 

### Using `hiera_include`

Where last we left off, our `site.pp` manifest was looking somewhat spare. With the `hiera_include` function, we can pare things down even further by picking a key to use for classes (we recommend `classes`), then declaring it in our `site.pp` manifest:

	hiera_include('classes')

From this point on, you can add or modify an existing Hiera data source to add an array of classes you'd like to assign to matching nodes. In the simplest case, we can visit each of kermit, grover, and snuffie and add this to their JSON data sources in `/etc/puppet/hiera/node`:

	"classes" : "ntp",

modifying kermit's data source, for instance, to look like this:

	{  
	   "classes" : "ntp",
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

`hiera_include` requires either a string with a single class, or an array of classes to apply to a given node:

	{  
	   "classes" : [
		   "ntp",
		   "apache",
		   "postfix"
	   ],
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

That demonstrates a very simple case for `hiera_include`, where we know that we want to assign a particular node to a specific host. But just as we used the `fqdn` fact to choose which of our nodes received specific parameter values, we can use that or any other fact to drive class assignments. Some organizations might choose to apply the `apache` class to nodes with a given value in their `hostname` fact (e.g. `www`), or assign a `vmware_tools` class that installs and configures VMWare Tools packages on every host that returns `vmware` as the value for its `virtual` fact. 

To configure that last case, for instance, we might change our `hiera.yaml` file to look like this:

	---
	:backends:
	  - json
	:json:
	  :datadir: /etc/puppet/hieradata
	:hierarchy:
	  - node/%{::fqdn}
      - virtual/%{::virtual}
	  - common

Then add a `vmware.json` file to the `/etc/puppet/hiera/virtual` directory that includes the following:

	{
	   "classes" : "vmware_tools"
	}




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




## Scraps 

> **JSON or YAML?** Out of the box, Hiera supports both YAML and JSON files as data sources. Both work fine, so choosing one is a question of personal preference. We went with JSON, but that doesn't constitute a recommendation. We do recommend you stick to one or the other for simplicity's sake.


Junior people: make a new node by duping the file


### Remove Conditional Logic For Facts We Don't Control

A third thing we can simplify is the conditional logic our ntp module has to include to deal with the many different names for packages, services, and configuration files assorted operating systems use. This is a problematic use case for Hiera, but you may find it useful in your own organization.

The ntp module has fairly modest goals: On four supported operating systems, it installs an ntp package, then configures and manages the ntp service. There are only 21 lines of code in the manifest that actually describe the needed configuration. But it includes [84 lines of conditional logic][ntp_module_conditional] to check for the operating system in use on a given node, identify the name of the ntp package and service, and identify a template file to use for configuration. By removing that conditional logic, we'll be able to reduce the size of the ntp module's `init.pp` manifest by almost half and we'll make it easier to add new operating systems to our network over time: Instead of rewriting conditional logic in the template, we can create a new Hiera data source. 

The downside of this kind of simplifcation is that it makes your Hiera data sources more complex and it makes your modules a little harder to share. Before we replace some of the conditional logic we find in the ntp module, we should keep in mind that:

- It will no longer be enough to use Hiera to cover the five user-facing parameters the module originally required: We'll also have to introduce data sources to cover the supported operating systems, and we'll probably end up scattering module-related data throughout several orthogonal elements of our hierarchy. We'll also need to turn a number of variables internal to the module --- variables the authors never intended to be user-facing --- into additional parameters for the `ntp` class the module provides. In some ways, the module will have to become more complex.

- If you stick to writing modules that don't _expect_ Hiera, your code will be much more easily shared and reusable: People who are using Hiera already will know exactly how to configure it to work with your module; people who aren't using Hiera won't have to learn just to use your code.

These are issues with Hiera as it currently exists, and [we're working on them][hiera_module_data_ticket]. 

We're still going to show how to move some of this conditional logic into Hiera, but we recommend it only in cases where you don't plan to share your module with others, and where you have a good plan for keeping up with the several places configuration data related to a given module may exist.
