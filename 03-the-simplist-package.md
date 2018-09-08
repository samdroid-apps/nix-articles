---
title: Creating a super simple derivation - Learning Nix pt 3
summary: Wrapping some shell scripts
$dates:
    published: 1 Febuary 2018
order: 204
hero: url(/static/images/204/hero.jpg)
thumb: /static/images/204/hero.jpg
---


This guide will build on the previous [two][part2] [guides][part1], and look at creating your first useful _derivation_ (or "package").

This will teach you how to package a shell script.

## Packaging a shell script (with no dependencies)

We can use the function `pkgs.writeShellScriptBin` from NixPkgs, which
handles generating a _derivation_ for us.

This function takes 2 arguments; what name you want the script to have in your PATH, and a string being the contents of the script.

So we could have:

```nix
pkgs.writeShellScriptBin "helloWorld" "echo Hello World"
```

That would create a shell script named "helloWorld", that printed "Hello World".

Let's put that in an environment; so we can use it in `nix-shell`.  Write this to
`test.nix`:

```nix
with import <nixpkgs> {};

let
  # Use the let-in clause to assign the derivation to a variable
  myScript = pkgs.writeShellScriptBin "helloWorld" "echo Hello World";
in
stdenv.mkDerivation rec {
  name = "test-environment";

  # Add the derivation to the PATH
  buildInputs = [ myScript ];
}
```

We can then enter the `nix-shell` and run it:

```sh
sam@vcs ~> nix-shell test.nix

[nix-shell:~]$ helloWorld
Hello World
```

Great!  You've successfully made your first package.  If you use NixOS, you can modify your system configuration and include it in your `environment.systemPackages` list.  Or you can use it in a `nix-shell` (like we just did).  Or whatever you want!  Despite being one line of code, this is a real Nix _derivation_ that we can use.

## Referencing other commands in your script

For this example/section; we are going to look at something more complex.  Say you want to write a script to find your public IP address.  We're basically going to run this command:

```sh
curl http://httpbin.org/get | jq --raw-output .origin
```

But running this requires dependencies; you need `curl` and `jq` installed.  How do we specify dependencies in Nix?

Well, we could just add them to the build input for the shell:

```nix
# DO NOT USE THIS; this is a BAD example
with import <nixpkgs> {};

let
  # This is the WORST way to do dependencies
  # We just specify the derivation the same way as before
  simplePackage = pkgs.writeShellScriptBin "whatIsMyIp" ''
    curl http://httpbin.org/get | jq --raw-output .origin
  '';
in
stdenv.mkDerivation rec {
  name = "test-environment";

  # Then we add curl & jq to the list of buildInputs for the shell
  # So curl and jq will be added to the PATH inside the shell
  buildInputs = [ simplePackage pkgs.jq pkgs.curl ];
}
```

This would work OK; you could go `nix-shell` then run `whatIsMyIp` and get your IP.

But it has a problem.  The script would work unpredictably.  If you took this package, and used it outside of the nix-shell, it wouldn't work - because you didn't have the dependencies.  It also pollutes the environment of the end user; as they need to have a compatible version jq and curl in their path.

The more eloquent way to do this is to reference the exact packages in the shell script:

```nix
with import <nixpkgs> {};

let
  # The ${...} is for string interpolation
  # The '' quotes are used for multi-line strings
  simplePackage = pkgs.writeShellScriptBin "whatIsMyIp" ''
    ${pkgs.curl}/bin/curl http://httpbin.org/get \
      | ${pkgs.jq}/bin/jq --raw-output .origin
  '';
in
stdenv.mkDerivation rec {
  name = "test-environment";

  buildInputs = [ simplePackage ];
}
```

Here we reference the dependency package inside the derivation.  To understand what this is doing, we need to see what the script is written to disk as.  You can do that by running:

```
sam@vcs ~> nix-shell test.nix

[nix-shell:~]$ cat $(which whatIsMyIp)
```

Which gives us:

```sh
#!/nix/store/hqi64wjn83nw4mnf9a5z9r4vmpl72j3r-bash-4.4-p12/bin/bash
/nix/store/pkc7g36m95jymw3ga2i7pwrykcfs78il-curl-7.57.0-bin/bin/curl http://httpbin.org/get \
  | /nix/store/znqn0z505i0bm1aiz2jaj1ki7z4ck1sv-jq-1.5/bin/jq --raw-output .origin
```

As we can see, all the binaries referenced in this script are absolute paths, something like `/nix/store/...../bin/name`.  The `/nix/store/...` is the path of the derivation's (package's) build output.

Due to the pure and functional of Nix, that path will be the same on every machine that ever runs Nix.  Replacing fuzzy references (eg. `jq`) with definitive and unambiguous ones (`/nix/store/...`) is a core tenant of Nix; as it means packages come will all their dependencies and don't pollute your environment.

Since it is an absolute path, that script doesn't rely on the PATH environment variable; so the script can be used anywhere.

When you reference the path (like `${pkgs.curl}` from above), Nix automatically knows to download the package into the machine whenever your package is downloaded.

> **Why do we do it like this?**  Ultimately, the goal of package management is to make consuming software easier.  Creating less dependencies on the environment that runs the package makes it easier to use the script.

So the TL;DR is:

```nix
# BAD; not very explicit
# - we need to remember to add curl to the environment again later
badPackage = pkgs.writeShellScriptBin "something" ''
  curl ...
'';

# GOOD: Nix will do the magic for us
goodPackage = pkgs.writeShellScriptBin "something" ''
  ${pkgs.curl}/bin/curl ...
'';
```

# Functions make creating packages easier

One of the main lessons from this process is that when you use functions (like `pkgs.writeShellScriptBin`) to create packages, it is pretty simple.  Compare this to a traditional RPM or DEB workflow; where you would have needed to write a long spec file, put the script in a separate file, and fight your way through too much boilerplate.

Luckily; NixPkgs (the standard library of packages) includes a whole raft of functions that make packaging easier for specific needs.  Most of these are in the [build support](https://github.com/NixOS/nixpkgs/tree/master/pkgs/build-support) folder of the NixPkgs repository.  These are defined in the Nix expression language; the same language you are learning to write.  For example, [the pkgs.writeShellScriptBin function](https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/trivial-builders.nix#L61) is defined as a ~10 line function.

Some of the more complex build support functions are documented in [the NixPkgs manual](https://nixos.org/nixpkgs/manual/#chap-language-support).  There is currently documentation for packaging Python, Go, Haskell, Qt, Rust, Perl, Node and many other types of applications.

Some of the more simple build support functions (like `pkgs.writeShellScriptBin`) are not documented (when I write this).  Most of them are self explanatory, and can be found by reading their names in the [so called _trivial builders_ file](https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/trivial-builders.nix).

## Up Next

[Derivations 102 - Learning Nix pt
4](https://www.sam.today/blog/derivations-102-learning-nix-pt-4)

Follow the series [on GitHub](https://github.com/samdroid-apps/nix-articles)

*Hero image from [nix-artwork by Eric Sagnes](https://github.com/NixOS/nixos-artwork/blob/master/wallpapers/nix-wallpaper-mosaic-blue.png)*

[part1]: https://www.sam.today/blog/environments-with-nix-shell-learning-nix-pt-1/ "Environments with Nix Shell - Learning Nix pt 1"
[part2]: https://www.sam.today/blog/so-variables-are-a-thing-learning-nix-pt-2/
