# mibed

## About

Mibed allows you to easily create SmartOS zone datasets by doing a git push from a mibe repository.
Datasets can automatically be published to a dsapi server.

Despite its name it is not an actual daemon but a set of shell scripts that work with the original [mibe](https://github.com/joyent/mibe).

## Screenshot

	# git push mibed master
	Counting objects: 5, done.
	Delta compression using up to 4 threads.
	Compressing objects: 100% (3/3), done.
	Writing objects: 100% (3/3), 287 bytes | 0 bytes/s, done.
	Total 3 (delta 2), reused 0 (delta 0)
	remote: Image dc0688b2-c677-11e3-90ac-13373101c543 (base64 13.4.2) is already installed, skipping
	remote: 
	remote: build_smartos - version 1.0.0
	remote: Image builder for SmartOS images
	remote: 
	remote: * Sanity checking build files and environment..                       OK.
	remote: * Halting build zone (9c0ff371-57f9-41b1)..                           OK.
	remote: * Configuring build zone (9c0ff371-57f9-41b1) to be imaged..          OK.
	remote: * Booting build zone (9c0ff371-57f9-41b1)..                           OK.
	remote: * Copying in mi-graphite-1360a6ba40c2234ce558a5311216d3906f515c97/copy files..OK.
	remote: * Creating image motd and product file..                              OK.
	remote: * Installing packages list..                                          OK.
	remote: * Executing the customize file..                                      OK.
	remote: * Halting build zone (9c0ff371-57f9-41b1)..                           OK.
	remote: * Un-configuring build zone (9c0ff371-57f9-41b1)..                    OK.
	remote: * Creating image file and manifest..                                  OK.
	remote: 
	remote: Image:    /opt/mibed/mibe/images/graphite-13.2.6.zfs.gz
	remote: Manifest: /opt/mibed/mibe/images/graphite-13.2.6.dsmanifest
	remote: 
	remote: Uploading to dsapid...
	remote: ######################################################################## 100.0%
	remote: 
	remote: UUID: 0cb4bc40-d79e-11e3-ae44-8ffa98d772ea
	remote: 
	To git@mibed.example.com:mi-graphite
	   764949b..1360a6b  master -> master

## /!\ Warning

The following installation steps will modify the the global zone.
This is usually discouraged with good reason.

It also creates a new "git" user which has "Primary Administrator" privileges.
While keys added with <code>/opt/mibed/bin/add-key</code> are limited to the gitreceive hook there might be ways to escape that!

It is recommended that you use a dedicated SmartOS system for mibed and only allow trusted users access to it.

# Installation

There is some preparation required which is also documented here.

## Setup a DSAPI Server

If you want to auto-publish Datasets after creation you need to setup a "**daspid**" Instance from <http://datasets.at>.
Inside the instance you'll want to add a new user to <code>/data/users.json</code> like:

	...
	{
		"uuid": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
		"name": "mibed",
		"type": "user",
		"provider": "testing",
		"token": "secrettoken",
		"roles": ["s_dataset.admin", "s_dataset.upload"]
	}
	...

You'll need the IP/Hostname of that instance and the token later.

## Prepare the Global Zone

### Install git

Git is required to accept remote pushes. It can be installed from pkgsrc.

If pkgsrc is not yet installed in the GZ it can be bootstraped with these commands:

	cd /
	curl -k http://pkgsrc.smartos.skylime.net/packages/SmartOS/bootstrap/bootstrap-2014Q1-x86_64.tar.gz | gzcat | tar -xf -
	echo http://pkgsrc.smartos.skylime.net/packages/SmartOS/2014Q1/x86_64/All/ > /opt/local/etc/pkgin/repositories.conf
	pkg_admin rebuild
	pkgin -y up

To install git use:

	pkgin install git-base

### Create buildzone

Mibe needs a zone to work its magic. Create one with:

	cat <<EOF > mibezone.json
	{
	  "brand": "joyent",
	  "image_uuid": "dc0688b2-c677-11e3-90ac-13373101c543",
	  "alias": "mibezone",
	  "hostname": "mibezone",
	  "max_physical_memory": 512,
	  "quota": 20,
	  "nics": [
	    {
	      "nic_tag": "admin",
	      "ip": "dhcp",
	      "primary": "true"
	    }
	  ]
	}
	EOF
	vmadm create -f mibezone.json

## Install mibed

Install mibed to <code>/opt</code>:

	cd /opt
	curl -Lk https://github.com/wiedi/mibed/archive/v0.1.tar.gz | gtar -zxf -
	mv mibed-0.1/ mibed
	/opt/mibed/bin/install

Edit <code>/opt/mibed/config</code> to configure the UUID of the buildzone and the connection details to the dsapid:

	MIBE_ZONE=9c0ff371-57f9-41b1-bff2-1883e2ac2b92
	DSAPID=http://secrettoken@datasets.example.com
	
# Usage

Add a new user like this:

	cat .ssh/id_rsa.pub | ssh mibed.example.com /opt/mibed/bin/add-key

Add the new remote to your mibe git repo (here mi-graphite):

	git remote add mibed git@mibed.example.com:mi-graphite

From now on you can trigger new builds with:

	git push mibed master

## Base Image

By default mibed will use the latest base64 image as a base. You can specifiy the UUID of a different base image in the manifest of your repository with the <code>base</code> variable.

# Thanks

- Joyent for [SmartOS](http://smartos.org) and [MIBE](https://github.com/joyent/mibe)
- MerlinDMC for [dsapid](https://github.com/MerlinDMC/dsapid) and [datasets.at](http://datasets.at)
- [gitreceive](https://github.com/progrium/gitreceive)
