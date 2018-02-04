# DRAFT: Derivations 102

Nix is built around the concept of derivations.  A derivation is simply defined as ["a build action"](https://nixos.org/nix/manual/#ssec-derivation).  It produces 1 (or maybe more) output paths in the nix store.

> AKA; a puts something in an output directory (that can be referenced later by it's unique nix store path)

It's important to note that literally everting in NixOS is built around derivations.  Applications?  Of course they are derivations.  Your configuration files?  They are built in a derivation and put in the nix store.  The `/run/current-system` directory on NixOs?

```nix
sam@vcs ~> ls -lsah /run/current-system
0 lrwxrwxrwx 1 root root 83 Jan 25 13:22 /run/current-system -> /nix/store/wb9fj59cgnjmkndkkngbwxwzj3msqk9c-nixos-system-vcs-17.09.2683.360089b3521
```

It's a symbolic link to a derivation!

It's derivations all the way down.

If you've been observant; you would have noticed that we've already made some derivations.  Nix-shell (in part 1) is based off having a derivation.  When packaging a shell script, we made a derivation (in part 3).

I think it is easiest to learn how to make a derivation through examples.  Most packaging tasks are vaguely similar to packaging tasks done in the past by other people.  So this will be going through example of using `mkDerivation`.

## mkDerivation

Making a derivation manually [requires mucking with things like architecture and having zero standard build-inputs](https://nixos.org/nix/manual/#ssec-derivation).  So instead, the NixPkgs function `stdenv.mkDerivation` is commonly used.

In real world usage usage, you need the following things:

* Your derivations has some folder of source material
  - Folders fetched from git, github, tarballs, etc. are OK
  - If you have no source, or just 1 file; consider the "traivial builders" covered in section 3

mkDerivations does a lot of shit automagically; so you should understand that is there.  There are the "phases" of the build; most of which have some default behaviour:

* **unpack**: unzips, untarz, or copies your source folder to the nix store
* **patch**: applies any patches provided in the `patches` variable
* **configure**: runs `./configure` if it exists
* **build**: runs `make` if it exists
* **check**: literally skipped by default
* **install**: runs `make install`
* **fixup**: does automitically fixes up things that don't jell with the nix store; such as using incorrect interpreter paths
* **installCheck**: runs `make installcheck` if it exists and is enabled

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

We can then use that derivation as a webroot in a nginx virtualHost;

```nix
let
  content = stdenv.mkDerivation { ... }
in
  services.nginx.virtualHosts."example.com" = {
    locations = {
      "/" = {
        root = "${content}";
      }
    }
  };
```

So how does this work?  Ultimatly, the "root" attribute needs to be set to the output directory of the content derivation.

Using the `"${content}"` expression, we force the derivation to be converted to a string (remembering `${...}` is string interpolation syntax).  When a derivation is converted to a string in Nix, it becomes the output path in the Nix store.

Or we can use it for a simple http server python script:

```nix
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

# Example #2: A more complex shell app

We've already talked a lot about shell scripts.  But sometimes whole apps get
built in shell scripts.  One such example is [emojify](https://github.com/mrowa44/emojify),
a CLI tool for replacing words with emojis.

We can make a derivation for that.  All we need to do is copy the shell script into the PATH, and mark it as executable.

If we were writing the script ourself, we'd need to pay special attention to fixing up dependancies (such as changing `/bin/bash` to a Nix store path).  But `mkDerivation` has the _fixup phase_, which does this automatically.  The defaults are smart, and in this case it works perfectly.

So we can go use code like:

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
        # aka. If it will always go the same because the source
        #      always has the same hash
        sha256 = "0zhbfxabgllpq3sy0pj5mm79l24vj1z10kyajc4n39yq8ibhq66j";
      };

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

# Example #3: The infamous GNU hello example

If you've ever read anything about Nix, you might have seen an example of making a derivation for GNU hello.  Something like this:

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

To understand it, we need to remember the default build phases from `stdenv.mkDerivtion`.  Here are the most important default phases to remember:

* **unpack**: unzips, untarz, or copies your source folder to the nix store
  - this is how the source is extracted
* **configure**: `./configure`
* **build**: `make`
* **check**: `make check`
* **install**: `make install`

Since GNU hello uses Make & `./configure`; the defaults work perfectly for us in this case.

## Your Packing Future

While it's amazing to use `mkDerivation` (so much easier than an RPM spec), there are many cases when you should _not_ use `mkDerivation`.  NixPkgs contains many useful "build support" functions.  These make it easy to build packages that meet specified crtiteria.

For example, there is `pkgs.python36Packages.buildPythonPackage`, which is a super easy way to build a python package.

When making packages, there are helpful resources to check:

* [Chapter 9. Support for specific programming languages and frameworks](https://nixos.org/nixpkgs/manual/#chap-language-support) of the NixPkgs manual.  This documents language specific build support functions.
* The [NixPkgs repository](https://github.com/nixOs/nixpkgs).  Many packages you make could be similar to packages that already exist.
  - Note that packages inside of NixPkgs are written in a bit of a different way to ours; since they can't reference NixPkgs directly.  Instead, they are structured as functions.  If a package depends on another, it takes the other packages as an argument.  For more on this subject; see [Nix Pill 13: Callpackage Design Patten](TODO LINK)
