# RabbitVCS Packaging Procedure

## Quick instructions

1. Set up a Debian build environment using Pbuilder or a tool like it.

```text
DIST=vivid sudo -E pbuilder create
```

2. Check out or create a branch off `master` named for the distribution you plan to package for, eg.

```text
cd dev/rabbitvcs-packaging/rabbitvcs
git checkout ubuntu-vivid
```

3. Use the Debian packaging tools to update the version number, changelog, and manually update any other configuration details that need to be updated.

```
dch -b -v 0.16-1~vivid "Packaging for new distro"
```

4. Build the binary package using Pbuilder/Git-buildpackage, using a command like:

```text
git-buildpackage --git-builder="pdebuild --buildresult /home/dev/rabbitvcs-build-area"
```

Check that everything is as it should be in the binary package (ie. internal paths are correct). Ensure it installs and works.

5. Build the signed source package using Pbuilder/Git-buildpackage, like:

```text
git-buildpackage --git-builder="debuild -S -sa"
```
 
6. Upload the signed source packages to the Launchpad PPA:

```text
dput rabbitvcs rabbitvcs_0.16-1~vivid_source.changes
```

## Debian and Ubuntu packaging

**Terminology:** I will refer to the packaging here as "Debian" packaging, even though it applies to both Ubuntu and Debian, and really I'm focusing on Ubuntu. For example, I might refer to `rabbitvcs/debian/changelog` as "the Debian changelog". This is because most of the documentation and tools were written for Debian, and for a lot of the packaging standards and details Debian is where things get tried and tested first.

The point of Debian packaging is to allow a package manager to "know" what software is installed on a system and where, and what files are owned by which packages. It means that users can install, uninstall, audit, and hack on software without permanently messing up their system.

It is important to understand this, because it's why packaging for Debian and Ubuntu can be so detailed and exacting. The entire process: obtaining the source code, patching, building, installing, and uninstalling should be completely automated and reversible.

The whole idea of Debian packaging is this: you take your "upstream" software that can be built and installed on any system. You add some files to it in a `debian/` subdirectory. These files tell Debian's specialised build tools and package manager what your software depends on, how to build it, and how to install it.

The `debian/` subdirectory should be the entire extent of this. In fact, it should be possible to distribute only that subdirectory and have the packaging work! The files in there say where upstream code lives, how to build it, whether it needs patching (those patches also live somewhere in `debian/`), etc.

## Pbuilder and repeatable build tools

Pbuilder is a tool for building your packages in a completely clean environment.

Have a read of some of these resources:

 * [User's manual](https://pbuilder.alioth.debian.org/)
 * [Pbuilder tricks](https://wiki.debian.org/PbuilderTricks) (what my `.pbuilderrc` is based on)
 * The Pbuilder man page

The point of a tool like Pbuilder is to ensure that you get your build dependencies right, and that you don't have odd workarounds for your build procedure that only work on your machine. If your package does not build under Pbuilder, it won't build on eg. Launchpad, or Debian's autobuild farm, or (more to the point) on any random user's machine if they want to hack on it.

You don't have to use Pbuilder for this, there is also [Sbuild](https://wiki.debian.org/sbuild). But I haven't used it, so I won't say anything more about it.

I will say, though, that you absolutely should be using one of these tools. Building packages straight from your dev machine, where you've got all of your personal hacks and libraries and build scripts installed, is a sure way to give yourself massive headaches later when Launchpad tells you your packages won't build.

### What Pbuilder generates

Pbuilder can produce two things: a source package and a binary package. The binary package is what a user installs to just use the software. The source package contains the source and information they need to hack on or build the software.

If you need to send someone something to install (eg. if they're testing something for you), send them the binary package ie. the `.deb` file. If you're uploading something to a PPA, it'll be the source packages ie. the `.dsc`, `.changes` and `.debian.tar.xz` files.

### Configuring Pbuilder

This repository should also contain a file named `dot_pbuilderrc`. This is my configuration for Pbuilder, which goes in `~/.pbuilderrc`. It contains code names for Debian and Ubuntu releases, some code for parsing Debian packaging files and deducing what release you're building for, and other useful time savers.

There are also some *hooks:* scripts that are executed at particular points in the build process. I have a hook to run Lintian (which checks for common errors in Debian packages) and another one that starts an interactive shell if a build fails. These are in `pbuilder-hooks/`.

You might like to change `BASEDIR="$HOME/pbuilder"` to a subdirectory if you want to keep your home directory clean eg. `BASEDIR="$HOME/Pbuilder/pbuilder"`. Note that this will change where your hooks go too.

If you're packaging for a new distribution, you will need to create a new Pbuilder environment. You do this with:

```text
DIST=vivid sudo -E pbuilder create
```

This creates a compressed archive of a bare-bones Debian (or Ubuntu) installation. I'll go into how to actually build the package later.

## Packaging details

Things to read:

 * The [Debian New Maintainers' Guide](https://www.debian.org/doc/manuals/maint-guide/) - this is a very good explainer of all the things I just said.
 * The [Debian policy manual](https://www.debian.org/doc/debian-policy/). This is a versioned, detailed document describing how Debian actually works as a distribution. Especially read:
   * The [source packages](https://www.debian.org/doc/debian-policy/ch-source.html) chapter.
   * The [binary packages](https://www.debian.org/doc/debian-policy/ch-binary.html) chapter.

The package is built via the `debian/rules` file, which is actually just a GNU Make makefile.

RabbitVCS' `rules` file is incredibly simple, because it's a straightforward Python application. Most of the configuration is done by the files in the `debian` directory eg. `debian/rabbitvcs-gedit.install` contains instructions for the Gedit plugins to be installed. The `control` file contains package management information: dependencies, descriptions, etc. It has multiple sections for all the different packages.

### Versioning

The versioning of the packages is a little complicated. There is the *upstream version* eg. `0.16`. The Debian package follows this after a hyphen, so you get `0.16-1`. But then Launchpad needs to be able to tell the packages for each distribution apart, so you need something like `0.16-1~vivid`. It's a bit of a pain, but a systematic pain.

### Standards versions

Remember how I said that the Debian policy manual is a versioned document? Well, the packaging files declare which version of that document they were written for (in `debian/control`). Lintian (which can be run with that hook script I mentioned) will warn you if the package lists an older standards version than what's currently released.

Don't go just bumping the standards version to silence this warning though. You should look at what's changed in the policy manual, decide whether it applies to your packaging or not, make the appropriate changes, and then update the version number when you've fixed anything that needs fixing.

### A note about the packaging changelog

The file `debian/changelog` is a required part of the packaging. But it's not a generic changelog for every new feature or bugfix in the software itself! It's the changelog specifically for packaging changes eg. if you update the package description, change dependencies, or if a plugin were deprecated or added and you needed to change the `.install` files. Don't go filling this with every little thing that happens to RabbitVCS, it's not the right place for it.

You should update this changelog with the `dch` (Debchange) command, eg.

```text
dch -v 0.16-1~vivid "Packaging for Ubuntu 15.04 Vivid Vervet"
```

## Building with Git-buildpackage

So that's the packaging itself, and traditionally, the Debian packaging was maintained by Debian maintainers, separately from the upstream project. So this is important: the upstream project should never, itself, contain a `debian` directory, otherwise there will be a collision when a Debian maintainer goes to package it.

But in our case, we maintain our own packaging. To avoid this paradox, we use a tool called `git-buildpackage`, which combines packaging information from specially named branches with the pristine upstream code in the main branch.

The RabbitVCS git repository has a number of branches including:

  * `debian-common`
  * `ubuntu-<name>` (where `<name>` is one of the Ubuntu distro names eg. `vivid`)

**Do not merge these with `master` or create pull requests from them!**

These branches contain the packaging information for each distribution. When you run `git-buildpackage`, it will merge the packaging branch with a tagged release to create the "Debianised" package, and then run the appropriate utilities to build the package. I will refer you to [Git-buildpackage's documentation](https://honk.sigxcpu.org/piki/projects/git-buildpackage/) to learn about the intricacies of this command.

There's also a branch called `debian-common` that contains most of the information that's the same between all of the packaging branches. So if eg. you need to update the package description for all of the Ubuntu packages, you can change it in the `debian-common` branch and merge from `debian-common` into the other branches.

At the moment, the actual-Debian packaging is an ancestor of all the Ubuntu packaging, so I've just left it as `debian-common`. But it might be wise to create a specific `debian-<name>` branch off `debian-common` if it ever starts diverging.

### Building packages

I use Git-buildpackage to actually do the builds (in combination with Pbuilder for the binary builds). You need to designate somewhere for the built packages to go, which I've called `/home/dev/rabbitvcs-build-area` in the examples below.

You can build two types of packages: binary and source. The binary package is the compiled software that's actually installable on a user's system via `dpkg` or the Software Centre. The source package is the zipped up `debian` directory and a bunch of signatures, so that PPAs and Debian repositories (and users who love source code) can build the binary packages themselves.

For binary packages, I check out the appropriate branch and do:

```text
git-buildpackage --git-builder="pdebuild --buildresult /home/dev/rabbitvcs-build-area"
```

For source packages:

```text
git-buildpackage --git-builder="debuild -S -sa"
```

### Uploading packages to the Lauchpad PPA

## Launchpad's requirements

Launchpad have a bunch of requirements for you to meet before you're allowed to upload packages to their PPA. I recommend you check out their [help section](https://help.launchpad.net/) and get yourself acquainted. There's a section [specifically on PPAs](https://help.launchpad.net/Packaging), but you also need to agree to their community guidelines, register your public key so packages can be verified, etc.

## Uploading with Dput

When the packaging is ready and the source package built, you need to upload it to the Launchpad PPA using the `dput` command. Dput's configuration file is `~/.dput.cf` and contains information about the PPAs you're uploading to. This repository contains my Dput config as `dot_dput.cf` that should work out of the box for RabbitVCS. `rabbitvcs` refers to the main PPA, but there is also a beta testing PPA labelled as `rabbitvcs-testing`

You run Dput on the `.changes` file like so:

```text
dput rabbitvcs rabbitvcs_0.16-1~vivid_source.changes
```

It should automatically upload the associated files as well.

Launchpad will send you emails telling you whether the files were successfully uploaded or not, and then successfully built or not. You can check on their progress manually by going to the [RabbitVCS PPA package details page](https://launchpad.net/~rabbitvcs/+archive/ubuntu/ppa/+packages) and looking at the **Build Status** column. (I got to that page via the [RabbitVCS project page](https://launchpad.net/~rabbitvcs), then through the [RabbitVCS PPA](https://launchpad.net/~rabbitvcs/+archive/ubuntu/ppa) link, then the [**View package details**](https://launchpad.net/~rabbitvcs/+archive/ubuntu/ppa/+packages) link).

Once the packages are built, they will be available for users to install.
