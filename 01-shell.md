---
$title@: Environments with Nix Shell - Learning Nix pt 1
summary: An introduction for how to run Nix code
$dates:
    published: 29 January 2018
$order: 201
hero: url(/static/images/201/hero.jpg)
thumb: /static/images/201/hero.jpg
---

To start with learning Nix; we need a way to experiment.  Nix is a programming language, so we need a way to run our programs.  Nix is also a package/environment management tool, so we need a way to test our environments.

## Using nix-shell

Nix-shell lets you open a shell in a new environment.

In Nix; an environment is a collection of *derevations* (aka packages) that are put into your PATH.  This is really useful in many circumstances:

1. **Collaboration**; you can just send the `.nix` file to a collaborator and they will have the same things installed
2. **Cleanliness**; stuff inside a `nix-shell` isn't installed in your main environment; so you don't have to worry about uninstalling stuff or causing conflicts with other packages you love
3. **Developing** things; it is easy to build your own packages and test them inside a shell

We can create a environment by creating a `.nix` file to define the environment.  Create a file called `test.nix`:

```nix
# This imports the nix package collection,
# so we can access the `pkgs` and `stdenv` variables
with import <nixpkgs> {};

# Make a new "derivation" that represents our shell
stdenv.mkDerivation {
  name = "my-environment";

  # The packages in the `buildInputs` list will be added to the PATH in our shell
  buildInputs = [
    # cowsay is an arbitary package
    # see https://nixos.org/nixos/packages.html to search for more
    pkgs.cowsay
  ];
}
```

Then we can test this.  Use `nix-shell test.nix` to enter the environment.  Then you can run `sl` to see how it is added to the PATH:

```sh
> nix-shell test.nix

[nix-shell:~]$ echo "welcome to the nix environment" | cowsay
 ________________________________
< welcome to the nix environment >
 --------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

Then you can leave the nix-shell by pressing Ctrl-D.  If you try and run cowsay outside your environment, it won't work:

```sh
> cowsay
The program ‘cowsay’ is currently not installed. You can install it by typing:
  nix-env -iA nixos.cowsay
```

> Note: If you have cowsay installed in your main environment; choose another package you don't have installed

So we've made our first `nix-shell`.  This allows us to create self-contained groups of packages (useful in and of itself).  But we've also run our own nix code, the `test.nix` file, which will come in handy in the future.

## Understanding: How does that nix code actually work?

Remember our `test.nix` file?

```nix
with import <nixpkgs> {};

stdenv.mkDerivation {
  name = "my-environment";

  buildInputs = [
    pkgs.cowsay
  ];
}
```

What does this file actually do?

Well the first line is an import statement.  We'll come back to exactly how it works later in the guide; or you will figure out once you've got a good grasp of the concepts.  For now, it is magic that you need to put at the top of every file OK.

Let's attack the body of the code:

```nix
stdenv.mkDerivation {
  name = "my-environment";

  buildInputs = [
    pkgs.cowsay
  ];
}
```

This is actually some code written in the Nix expression language.  First let's learn some basic syntax.  I've put some similar examples in python to help illustrate the syntax:

| Syntax type | Python example | Nix expression language example |
|-------------|----------------|---------------------------------|
| Function calling | `function(some_value)` | `function some_value` |
| *Sets* (aka hashmaps, dictionaries) | `{"a": "b", "key": value}` | `{ a = "b"; key = value; }` |
| Lists | `[a, b, c]` | `[a b c]` |
| Accessing values of objects | sometimes `obj['key']`, others `obj.key` | `obj.key` |

So we can see our code calls `stdenv.mkDerivation`, and provides a *set* (dictionary) as the argument.
The set has the keys `name` and `buildInputs`.  These are used by the `stdenv.mkDerivation` function.

So what does `mkDerivation` do?
[Reading the documentation](https://nixos.org/nixpkgs/manual/#sec-using-stdenv), mkDerivation returns a derivation value.  A derivation simply represents anything that can be built; like a `package` but more generalized.

Since `mkDerivation` returns a value, our whole file returns a value when it is evaluated.  You can test this by printing the evaluated value:

```sh
> nix-instantiate --eval test.nix
{ __ignoreNulls = true; all = <CODE>; args = <CODE>; buildInputs = <CODE>; builder = <CODE>; ...
```

So this value is then used by the `nix-shell` program, and hey presto: we have a new environment.

## Extension: the shellHook attribute

When we are making our derivation for our environment, we can pass another useful value to the `mkDerivation` function.  This is the `shellHook`:


```nix
with import <nixpkgs> {};

stdenv.mkDerivation {
  name = "my-environment";

  buildInputs = [
    pkgs.figlet
    pkgs.lolcat
  ];

  # The '' quotes are 2 single quote characters
  # They are used for multi-line strings
  shellHook = ''
    figlet "Welcome!" | lolcat --freq 0.5
  '';
}
```

The `shellHook` value is shell code that that will be run when starting the iterative shell.

Running that example would result in an awesome welcome message:

```
> nix-shell test.nix
__        __   _                          _
\ \      / /__| | ___ ___  _ __ ___   ___| |
 \ \ /\ / / _ \ |/ __/ _ \| '_ ` _ \ / _ \ |
  \ V  V /  __/ | (_| (_) | | | | | |  __/_|
   \_/\_/ \___|_|\___\___/|_| |_| |_|\___(_)


[nix-shell:~]$
```

The shellHook property is very useful for setting environment variables and the like.

## Example Usage: python virtualenv on steroids

We can actually use this to make development environments when writing applications.  For example, say I'm developing a Python3 Flask application, but need the ffmpeg binary installed for the app to process some videos.  With virtualenv, you can't specify all the binary dependencies.  With Nix, you can use this `.nix` file:

```nix
with import <nixpkgs> {};

stdenv.mkDerivation rec {
  name = "python-environment";

  buildInputs = [ pkgs.python36 pkgs.python36Packages.flask pkgs.ffmpeg ];

  shellHook = ''
    export FLASK_DEBUG=1
    export FLASK_APP="main.py"

    export API_KEY="some secret key"
  '';
}
```

That simply combines our knowledge from before.  It gives me a shell with the
packages I request (python3.6, flask and ffmpeg) inside the PATH and PYTHONPATH.  It then runs the shellHook and sets the extra environment variables (like API_KEY) that my application needs to run.

## Up Next

So Variables are a Thing - Learning Nix pt 2

Follow the series [on GitHub](https://github.com/samdroid-apps/nix-articles)

*Hero image from [nix-artwork by Luca Bruno](https://github.com/NixOS/nixos-artwork/blob/master/gnome/Gnome_Dark.png)*
