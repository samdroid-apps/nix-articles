---
$title@: Derivations 102 - Learning Nix pt 4
summary: Taking advantage of the fact Nix is a programming language
$dates:
    published: 07 September 2018
$order: 205
hero: url(/static/images/203/hero.jpg)
thumb: /static/images/203/hero.jpg
---

This guide will build on the [previous][part1] [three][part2] [guides][part3], and look at creating a wider variety of useful nix packages.

Nix is built around the concept of derivations.  A derivation is simply defined as ["a build action"](https://nixos.org/nix/manual/#ssec-derivation).  It produces 1 (or maybe more) output paths in the nix store.

Basically, a derivation is a pure function that takes some inputs (dependencies, source code, etc.) and makes some output (binaries, assets, etc.).  These outputs are referenceable by their unique nix-store path.

## Derivation Examples

It's important to note that literally everting in NixOS is built around derivations:

* Applications?  Of course they are derivations.
* Configuration files?  In NixOS, they are a derivation that takes the nix configuration and outputs an appropriate config file for the application.
* The system configuration as a whole (`/run/current-system`)?

```sh
sam@vcs ~> ls -lsah /run/current-system
0 lrwxrwxrwx 1 root root 83 Jan 25 13:22 /run/current-system -> /nix/store/wb9fj59cgnjmkndkkngbwxwzj3msqk9c-nixos-system-vcs-17.09.2683.360089b3521
```

> It's a symbolic link to a derivation!

It's derivations all the way down.

If you've followed this series [from the beginning][part1], you would have noticed that we've already made some derivations.  Our [`nix-shell` scripts][part1] are based off having a derivation.  When [packaging a shell script][part3], we also made a derivation.

I think it is easiest to learn how to make a derivation through examples.  Most packaging tasks are vaguely similar to packaging tasks done in the past by other people.  So this will be going through example of using `mkDerivation`.

## mkDerivation

Making a derivation manually [requires fussing with things like processor architecture and having zero standard build-inputs](https://nixos.org/nix/manual/#ssec-derivation).  This is often not necessary.  So instead, NixPkgs provides a function function `stdenv.mkDerivation`; which handles the common patterns.

The only real requirement to use `mkDerivation` is that you have some folder of source material.  This can be a reference to a local folder, or something fetched from the internet by another nix function.  If you have no source, or just 1 file; consider the "trivial builders" covered in [part three of this series][part3]

`mkDerivation` does most a lot of work automatically.  It divides the build up into "phases", all of which include a little bit of default behaviour - although it is usually unintrusive or can be can be overridden.  The most important phases are:

1. **unpack**: unzips, untarz, or copies your source folder to the nix store
2. **patch**: applies any patches provided in the `patches` variable
3. **configure**: runs `./configure` if it exists
4. **build**: runs `make` if it exists
5. **check**: skipped by default
6. **install**: runs `make install`
7. **fixup**: automagically fixes up things that don't jell with the nix store; such as using incorrect interpreter paths
8. **installCheck**: runs `make installcheck` if it exists and is enabled

You can [see all the phases in the docs](https://nixos.org/nixpkgs/manual/#sec-stdenv-phases).  But with a bit of practice from the examples below you'll likely get the feel for how this works quickly.

# Example #1: A static site

Nix makes writing packages really easy; and with NixOps (which we'll learn later) Nix derivations are automagiaclly built and deployed.

First we need to answer the question of how we would build the static site ourself.  This is a jekyll site, so you'd run the jekyll command

```nix
with import <nixpkgs> {};

stdenv.mkDerivation {
  name = "example-website-content";

  # fetchFromGitHub is a build support function that fetches a GitHub
  # repository and extracts into a directory; so we can use it
  # fetchFromGithub is actually a derivation itself :)
  src = fetchFromGitHub {
    owner = "jekyll";
    repo = "example";
    rev = "5eb1b902ca3bda6f4b50d4cfcdc7bc0097bac4b7";
    sha256 = "1jw35hmgx2gsaj2ad5f9d9ks4yh601wsxwnb17pmb9j02hl3vgdm";
  };
  # the src can also be a local folder, like:
  # src = /home/sam/my-site;

  # This overrides the shell code that is run during the installPhase.
  # By default; this runs `make install`.
  # The install phase will fail if there is no makefile; so it is the
  # best choice to replace with our custom code.
  installPhase = ''
    # Build the site to the $out directory
    export JEKYLL_ENV=production
    ${pkgs.jekyll}/bin/jekyll build --destination $out
  '';
}
```

Now we can see that this derivation builds the site.  If you save it to `test.nix`, you can trigger a build by running:

```sh
> nix-build test.nix
/nix/store/b8wxbwrvxk8dfpyk8mqg8iqhp7j2c9bs-example-website-content
```

The path printed by `nix-build` is where `$out` was in the Nix store.  Your path might be a little different; if you are running a different version of NixPkgs, then the build inputs are different.

We can see the site has built successfully by entering that directory:

```sh
> ls /nix/store/b8wxbwrvxk8dfpyk8mqg8iqhp7j2c9bs-example-website-content
2014  about  css  feed.xml  index.html  LICENSE  README.md
```

### Using the content

We can then use that derivation as a webroot in a nginx virtualHost.  If you have a server, you could add the following to your NixOS configuration:

```nix
let
  content = stdenv.mkDerivation {
  name = "example-website-content";

    ... # code from above snipped
  }
in
  services.nginx.virtualHosts."example.com" = {
    locations = {
      "/" = {
        root = "${content}";
      }
    }
  };
```

So how does this work?  Ultimately, the "root" attribute needs to be set to the output directory of the content derivation.

Using the `"${content}"` expression, we force the derivation to be converted to a string (remembering `${...}` is string interpolation syntax).  When a derivation is converted to a string in Nix, it becomes the output path in the Nix store.

If you don't have a server handy, we can use the content in this a simple http server script:

```nix
# server.nix
with import <nixpkgs> {};

let
  content = stdenv.mkDerivation {
    name = "example-website-content";

    src = fetchFromGitHub {
      owner = "jekyll";
      repo = "example";
      rev = "5eb1b902ca3bda6f4b50d4cfcdc7bc0097bac4b7";
      sha256 = "1jw35hmgx2gsaj2ad5f9d9ks4yh601wsxwnb17pmb9j02hl3vgdm";
    };

    installPhase = ''
      export JEKYLL_ENV=production
      # The site expects to be served as http://hostname/example/...
      ${pkgs.jekyll}/bin/jekyll build --destination $out/example
    '';
  };
in
let
  serveSite = pkgs.writeShellScriptBin "serveSite" ''
    # -F = do not fork
    # -p = port
    # -r = content root
    echo "Running server: visit http://localhost:8000/example/index.html"
    # See how we reference the content derivation by `${content}`
    ${webfs}/bin/webfsd -F -p 8000 -r ${content}
  '';
in
stdenv.mkDerivation {
  name = "server-environment";
  # Kind of evil shellHook - you don't get a shell you just get my site
  shellHook = ''
    ${serveSite}/bin/serveSite
  '';
}
```

Then run `nix-shell server.nix`, you'll then start the server and can view the site!


# Example #2: A more complex shell app

We've already talked a lot about shell scripts.  But sometimes whole apps get
built in shell scripts.  One such example is [emojify](https://github.com/mrowa44/emojify),
a CLI tool for replacing words with emojis.

We can make a derivation for that.  All we need to do is copy the shell script into the PATH, and mark it as executable.

If we were writing the script ourself, we'd need to pay special attention to fixing up dependencies (such as changing `/bin/bash` to a Nix store path).  But `mkDerivation` has the _fixup phase_, which does this automatically.  The defaults are smart, and in this case it works perfectly.

It is quite simple to write a derivation for a shell script.

```nix
with import <nixpkgs> {};

let
  emojify = let
    version = "2.0.0";
  in
    stdenv.mkDerivation {
      name = "emojify-${version}";

      # Using this build support function to fetch it from github
      src = fetchFromGitHub {
        owner = "mrowa44";
        repo = "emojify";
        # The git tag to fetch
        rev = "${version}";
        # Hashes must be specified so that the build is purely functional
        sha256 = "0zhbfxabgllpq3sy0pj5mm79l24vj1z10kyajc4n39yq8ibhq66j";
      };

      # We override the install phase, as the emojify project doesn't use make
      installPhase = ''
        # Make the output directory
        mkdir -p $out/bin

        # Copy the script there and make it executable
        cp emojify $out/bin/
        chmod +x $out/bin/emojify
      '';
    };
in
stdenv.mkDerivation {
  name = "emojify-environment";
  buildInputs = [ emojify ];
}
```

And see it in action:

```sh
> nix-shell test.nix

[nix-shell:~]$ emojify "Hello world :smile:"
Hello world 😄
```

# Example #3: The infamous GNU Hello example

If you've ever read anything about Nix, you might have seen an example of making a derivation for GNU Hello.  Something like this:

```nix
with import <nixpkgs> {};

let
  # Let's separate the version number so we can update it easily in the future
  version = "2.10";

  # Now define the derivation for the app
  helloApp = stdenv.mkDerivation {
    # String interpolation to include the version number in the name
    # Including a version in the name is idiomatic
    name = "hello-${version}";

    # fetchurl is a build support again; and does some funky stuff to support
    # selecting from a predefined set of mirrors
    src = fetchurl {
      url = "mirror://gnu/hello/hello-${version}.tar";
      sha256 = "0ssi1wpaf7plaswqqjwigppsg5fyh99vdlb9kzl7c9lng89ndq1i";
    };

    # Will run `make check`
    doCheck = true;
  };
in
# Make an environment for nix-shell
stdenv.mkDerivation {
  name = "hello-environment";
  buildInputs = [ helloApp ];
}
```

You can build and run this:

```sh
> nix-shell test.nix

[nix-shell:~]$ hello
Hello, world!
```

Ultimately this is a terrible and indirect example.  This doesn't explicitly specify anything that the builder will actually run!  It really confused me when I was learning Nix.

To understand it, we need to remember the default build phases from `stdenv.mkDerivtion`.  From above, we had a list of the most important phases.  If we annotate the defaults with what happens in the case of GNU Hello, things start to make sense:

| | Phase | Default Behaviour | Behaviour with GNU Hello |
|-|-------|---------|-----------|
|1|**unpack**|unzips, untarz, or copies your source folder to the nix store|the source is a tarball, so it is automatically extracted|
|2|**patch**|applies any patches provided in the `patches` variable| nothing happens |
|3|**configure**|runs `./configure` if it exists| runs `./configure` |
|4|**build**| runs `make` if it exists| runs `make`, the app is built |
|5|**check**| skipped by default | we turn it on, so it runs `make check` |
|6|**install**| runs `make install` | runs `make install` |

Since GNU Hello uses Make & `./configure`, the defaults work perfectly for us in this case.  That is why this GNU Hello example is so short!

## Your Packing Future

While it's amazing to use `mkDerivation` (so much easier than an RPM spec), there are many cases when you should _not_ use `mkDerivation`.  NixPkgs contains many useful _build support_ functions.  These are functions that return derivations, but do a bit of the hard work and boilerplate for you.  These make it easy to build packages that meet specified criteria.

We've seen a few _build support_ today; such as `fetchFromGitHub` or `fetchurl`.  These just functions that return derivations.  In these cases, they return derivations to download and extract the source files.

For example, there is `pkgs.python36Packages.buildPythonPackage`, which is a super easy way to build a python package.

When making packages, there are helpful resources to check:

* [Chapter 9. Support for specific programming languages and frameworks](https://nixos.org/nixpkgs/manual/#chap-language-support) of the NixPkgs manual.  This documents language specific _build support_ functions.
* The [NixPkgs repository](https://github.com/nixOs/nixpkgs).  Many packages you make could be similar to packages that already exist.  (Note that packages inside of NixPkgs are written in a bit of a different way to ours; since they can't reference NixPkgs directly.  Instead, they are structured as functions.  If a package depends on another, it takes the other packages as an argument.  For more on this subject; see [Nix Pill 13: Callpackage Design Patten](https://nixos.org/nixos/nix-pills/callpackage-design-pattern.html))

## Up Next

In part 5, we'll learn about functions in the Nix programming language.  With the knowledge of functions, we can write go on and write our own _build support_ function!

Follow the series [on GitHub](https://github.com/samdroid-apps/nix-articles)

*Hero image from [nix-artwork by Luca Bruno](https://github.com/NixOS/nixos-artwork/blob/master/gnome/Gnome_Dark.png)*



[part1]: https://www.sam.today/blog/environments-with-nix-shell-learning-nix-pt-1/ "Environments with Nix Shell - Learning Nix pt 1"
[part2]: https://www.sam.today/blog/so-variables-are-a-thing-learning-nix-pt-2/
[part3]: https://www.sam.today/blog/creating-a-super-simple-derivation-learning-nix-pt-3/ "Creating a super simple derivation - Learning Nix pt 3"
