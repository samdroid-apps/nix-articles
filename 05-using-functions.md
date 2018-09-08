TODO: 

* put a simple example of using let-in inside a function at the top
* put in a conclusion
* emphasise that example 3 and 4 are from my actual configuration

# Grokking Nix functions with real usage examples

Functions are pretty important in programming.  They help make things less repetitive.

## Defining a function in Nix

Nix functions are the most simple thing.

Every function takes exactly one argument.  The syntax is:

```nix
argumentName : expression
```

And the function returns the result of _expression_.

Functions are just values in Nix, so you often need to use this in conjunction with a let-in expression.  Eg:

```nix
# test.nix
let
  addOne = x : x + 1;
in
  addOne 4
```

We can evaluate that code, and we've made our first function:

```sh
> nix-instantiate --eval test.nix
5
```

## Taking multiple arguments

So the functions in Nix are too simple to accept more than one argument.

But not all is lost!  What we can do is create closures.  I'll start with a bit of python that explains what we want to do:

```python3
def add(a):
  def add_inner(b):
    return a + b
  return add_inner

print(add(2)(3))
```

So we have a function that takes the first argument, then returns another function.  In python (and many other languages), that looks super ugly.  But in Nix, this looks very nice:

```nix
# test.nix
let
  add = a : b : a + b;
in
  add 2 3
```

We we can run to the predictable result:

```sh
> nix-instantiate --eval test.nix
5
```

If that code looks a little confusing, just add parenthesis in the order Nix evaluates the code:

```nix
let
  add = a : (b : (a + b));
in
  ((add 2) 3)
```

Here Nix would go from the inner-most parenthesis outwards; just like in basic algebra.  This is exactly the same as the python code.

## Example 1: Making lolcatify

Let's move on to a super useful example.  Say you _need_ to automatically created versions of a program with rainbow colored output.  Let's call this _lolcatifying_ (because we are using the rainbow colouring program [lolcat](https://github.com/busyloop/lolcat)).

To do this, we will create a shell script wrapping the program, and then pipe the output into `lolcat.  We're going to use the `pkgs.writeShellScriptBin` helper function ([as we introduced in part 3][part3]):

```nix
# test.nix
with import <nixpkgs> {};

let
  # Here we define our function to "lolcatify" a program.
  # It takes 2 arguments, the `package` of the app
  # and the `name` of original name the binary.
  lolcatify = package : name : pkgs.writeShellScriptBin "lol-${name}" ''
    # shell script to pass all argument to the original command,
    # and pass output to lolcat
    ${package}/bin/${name} "$@" | ${pkgs.lolcat}/bin/lolcat
  '';
in
stdenv.mkDerivation rec {
  name = "lol-environment";
  # Just define a standard kind of nix-shell environment with some
  # example lolcatified binaries

  buildInputs = [
    # We must use parenthesis to avoid confusing between
    # a function call (what we want) and list items (which are
    # also space separated)
    (lolcatify pkgs.figlet "figlet")
    (lolcatify pkgs.cowsay "cowsay")
  ];
}
```

We can run this, and see the rainbow text.  You need to run it youself;
because I don't know how to copy and paste rainbow text:

```sh
> nix-shell test.nix

[nix-shell:~]$ lol-cowsay Hello world
 _____________
< Hello world >
 -------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||

[nix-shell:~]$ lol-figlet Text
 _____         _
|_   _|____  _| |_
  | |/ _ \ \/ / __|
  | |  __/>  <| |_
  |_|\___/_/\_\\__|
```

As an exercise, you could consider creating a similar function _cowsayify_;
which uses cowsay instead of lolcat.

Basically, what we have made is a build support function.  It lets us make
certian kinds of packages very eaisly.  While lolcatifying programs isn't a
common use case, this can be applied more widely.

## Taking loads of arguments

### Exposition

So lolcat has a bunch of arguments:

```sh
> lolcat --help

Usage: lolcat [OPTION]... [FILE]...

Concatenate FILE(s), or standard input, to standard output.
With no FILE, or when FILE is -, read standard input.

    --spread, -p <f>:   Rainbow spread (default: 3.0)
      --freq, -F <f>:   Rainbow frequency (default: 0.1)
      --seed, -S <i>:   Rainbow seed, 0 = random (default: 0)
       --animate, -a:   Enable psychedelics
  --duration, -d <i>:   Animation duration (default: 12)
     --speed, -s <f>:   Animation speed (default: 20.0)
         --force, -f:   Force color even when stdout is not a tty
       --version, -v:   Print version and exit
          --help, -h:   Show this message
```

As the business needs of LolCo change, we now need to make wrappers that set specific lolcat options.
For example, `lol-cow` must be run with `--freq 0.5`.

But not all users of lolcatify want to override the options.  So we want to
change our build support function to be more eleoquent - we want to add options
with default parameters.

### Sets as arguments

This is actually a common problem; many functions in nix take a huge amount of arguments.

Think about our favourite function: `stdenv.mkDerivation`.  There are [tonnes of ways you can configure that function](https://nixos.org/nixpkgs/manual/#sec-stdenv-phases), but if you don't have to set every argument every time you use that function.  This is done by passing a set (also called dictionary or hasmap in other programming languages) to the `mkDerivation` function, and default values are used for the unset values.

This is obviously a common need; so Nix has some syntatic sugar for this case.

To accept a set, you can do:

```nix
# test.nix
let
  add = { a, b, c } : a + b + c;
in
  add { a = 1; b = 2; c = 3; }
```

This function performs like you would expect:

```sh
> nix-instantiate --eval test.nix
6
```

So when a function accepts a set of values, you can specify them in a comma separated fashion.

To use default values, you use the `?` syntax:

```nix
# test.nix
let
  # a is required
  # b & c default to 0
  add = { a, b ? 0, c ? 0 } : a + b + c;
in
  add { a = 1; b = 2; }
```

Which gives us a predicable result:

```sh
> nix-instantiate --eval test.nix
3
```

## Example 2: Lolcatify with default arguments

So let's take some optional parameters for settings things like the frequency, spread and seed.  These will then be passed to lolcat as command line arguments.  So here is an example:

```nix
with import <nixpkgs> {};

let
  # Here we define our function to "lolcatify" a program
  # It requires the binary name
  # It needs the package the binary comes from; defaulting to coreutils
  # It accepts other arguments or the lolcat command
  lolcatify = {
    name,
    package ? pkgs.coreutils,
    # Nix does not have floating point or decimal numbers
    frequency ? "0.1",
    spread ? "3.0",
    seed ? "0",
  } : let
    # Using a let-in clause is allowed in a function
    # remember is is just an expression after all
    lolcatArgs = "--freq ${frequency} --spread ${spread} --seed ${seed}";
  in
    pkgs.writeShellScriptBin "lol-${name}" ''
      # shell script to pass all argument to the original command,
      # and pass output to lolcat
      ${package}/bin/${name} "$@" | ${lolcat}/bin/lolcat ${lolcatArgs}
    '';
in
stdenv.mkDerivation rec {
  name = "lol-environment";

  buildInputs = [
    (lolcatify { name = "figlet"; package = pkgs.figlet; })
    # Changing the optional freq argument
    (lolcatify { name = "cowsay"; package = pkgs.cowsay; frequency = "0.5"; })

    # These use the default value for the package argument
    (lolcatify { name = "cat"; })
    (lolcatify { name = "seq"; })
  ];
}
```

You can then run it in nix-shell, and the `lol-cowsay` command will now have a more frequent rainbow!  You get the picture.

While `lolcatify` is not a very sensible function, it mirrors a common pattern: returning a derivation.  In effect, we created our own "build support" function.  Maybe wrapping a script in lolcat isn't very useful, it is very similar to wrapping a script for logging or monitoring purposes - one real world use case of a function like this.

## Example #3: Making Nginx configuration easier

NixOS already offers a Nix-style confirmation generator for Nginx, where you can set the [`services.nginx.*` options](https://nixos.org/nixos/options.html#services.nginx) and a proper configuration file will be generated for nginx.

So, you might configure a static site to virtual host with something like:

```nix
services.nginx.virtualHosts."example.com" = {
  forceSSL = true;
  enableACME = true;
  locations = {
    "/" = {
      root = "${staticContent}";
      extraConfig = "expires $expires_aggressive;";
    };
  };
};
```

But here is the thing.  This can get really repetitive if you have many static sites.  A few functions come in really useful here:

```nix
services.nginx.virtualHosts = let
  # The `base` function takes a given `locations` configuration,
  # and adds some shared virtualHosts options (for TLS)
  base = locations : {
    forceSSL = true;
    enableACME = true;
    inherit locations;
    # `inherit x;` just means `x = x;`
  };

  # Configuration for a static site; uses the base
  # function internally and makes the location block
  static = {
    dir,
    cacheAggressively ? true,
  } : base {
    # This is the locations block we are passing to the `base` function
    "/" = {
      root = dir;
      # Nix has if statements
      #     if x then a else b
      # Sorry for not introducting them earlier
      extraConfig = if aggressive then ''expires $expires_aggressive;'' else ""; 
  };
in {
  # Use the functions for more simple configuration
  "example.com" = static { dir = "${exampleStaticSiteContent}" };
  "other.example.com" = static { dir = "something"; cacheAggressively = false; };
}
```

So by using function, we move the common configuration into a common function,
and we can separate the needs (how other.example.com doesn't use aggressive caching)
from the implementation details (setting the `extraConfig`).

Sorry about these examples; they aren't as easy as `nix-shell` to interact with.  If you are running NixOS, you can put these in your system configuration.  Or spin up a server and try it out!

## Example #4: Easy backup script configuration

One part of my backup system is syncing directories on my server to a local
backup server.  For this, I run a systemd timer service on my local server that
rsyncs the files from the real server.

I need a few different backup timers running; but the code defining them
is mostly the same.  Only the backup target and source really changes.

So a Nix function is the easy way to solve this problem:

```nix
let
  # This function returns a systemd service
  # See https://nixos.org/nixos/options.html#systemd.services.%3Cname%3E for available options
  syncTimerUnit = {
    username, # username to login to the other server as
    src,      # source directory
    name,     # name of this backup job
    frequent ? true,  # if the job should be run frequently, defaults to true
  }: {
    description = "Sync ${name} (from ${src} via ${username})";

    # 8x per hour if frequent, else hourly
    startAt = if frequent then "*:0,7,15,22,30,37,45,52" else "hourly";
    after = [ "network.target" ];

    # add openssh to the path; otherwise rsync can not ssh
    path = [ pkgs.rsync pkgs.openssh ];
    serviceConfig = {
      User = "root";
      # specifying the commnand line to run the app
      ExecStart = "${pkgs.rsync}/bin/rsync -a -r ${username}@server:${src} /backup/${name}";
    };
  };
in
{
  # ... other NixOS configuration omitted ...

  systemd.services.syncMaildir = syncTimerUnit {
    username = "virtualMail";
    src = "/var/lib/virtualMail";
    name = "virtualMail";
  };
  systemd.services.syncDKIM = syncTimerUnit {
    username = "root";
    src = "/var/lib/dkim";
    name = "dkim";
    frequent = false;
  };
  systemd.services.syncPostgresWalE = syncTimerUnit {
    username = "postgres";
    src = "/var/WALE";
    name = "wal_e";
  };
}
```

# Conclusion



[part3]: https://www.sam.today/blog/creating-a-super-simple-derivation-learning-nix-pt-3/ "Creating a super simple derivation - Learning Nix pt 3"
