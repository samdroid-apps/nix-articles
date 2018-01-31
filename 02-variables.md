---
$title@: So Variables are a Thing - Learning Nix pt 2
summary: Taking advantage of the fact Nix is a programming language
$dates:
    published: 31 January 2018
$order: 203
hero: url(/static/images/203/hero.jpg)
thumb: /static/images/203/hero.jpg
---

So Nix is fundamentally built around the Nix expression language; which is a programming language.  Creating variables is a huge part of programming.

If you want to package apps or just simplify repetitive configuration files; you will probably need variables.

## The let-in syntax

The let-lets syntax allows you you define a variable that the next expression of code runs in:

A high level example is:

```nix
let
  x = 1;
  y = 2;
in
  x + y
```

This is valid nix code; we can actually run it, if you save it to `test.nix`:

```sh
> nix-instantiate --eval test.nix
3
```

We can see here that the code still evaluates (aka. returns) the value 3.  This means that anywhere in our code, we can replace some `expression` with something like `let ... in expression`.

Here is a concrete example of that replacement.  We could make our last example more complex by replacing the number 1 with a let-in expression:

```nix
let
  x = (let a = 2; in a+3);
  y = 2;
in
  x + y
```

Which changes the answer:

```sh
> nix-instantiate --eval test.nix
7
```

So we can formalize the let-in syntax as:

```nix
let
  name = expression;
  name = expression;
  name = expression;
  ...
in
  expression
```

## Real world examples of the "let-in" expression

Say we have an environment (something we run as `nix-shell test.nix`); and it uses a lot of python packages:

```nix
with import <nixpkgs> {};

stdenv.mkDerivation rec {
  name = "python-environment";

  buildInputs = [
    pkgs.python36
    pkgs.python36.pkgs.flask
    pkgs.python36.pkgs.itsdangerous
    pkgs.python36.pkgs.six
  ];
}
```

Obviously that looks very repetitive; and we repeat the python version many times.

We can refactor this to store the `pkgs.python36` as a variable.  This makes the code less repetitive.  It also makes it easier to change the python version later.  The code would look like:

```nix
with import <nixpkgs> {};

stdenv.mkDerivation rec {
  name = "python-environment";

  buildInputs = let
    py = pkgs.python36;
  in [
    py
    py.pkgs.flask
    py.pkgs.itsdangerous
    py.pkgs.six
  ];
}
```

Yay!  Now we've created an identical environment with less words

### A digression on scope

Smart cookies reading along would have noticed that we could have
put the let-in expression in a different place.  For example:

```nix
with import <nixpkgs> {};

let
  # Assigning the variable `py` to the python we want to use
  py = pkgs.python36;
  # You could try and change it to python27 to see what happens
in
stdenv.mkDerivation rec {
  name = "python-environment";

  buildInputs = [
    # here we reference `py` rather than `pkgs.python36`
    py
    py.pkgs.flask
    py.pkgs.itsdangerous
    py.pkgs.six
  ];
}
```

The result of that code would have been identical.

However, putting the let-in expression in a different place changes the scope (or parts of the code) that the `py` variable is useable for.

So with the larger scope, we could do something like:

```nix
with import <nixpkgs> {};

let
  py = pkgs.python36;
in
stdenv.mkDerivation rec {
  name = "python-environment";

  buildInputs = [
    py
    py.pkgs.flask
    py.pkgs.itsdangerous
    py.pkgs.six
  ];

  # The ${...} syntax is string interpolation in Nix
  shellHook = ''
    echo "using python: ${py.name}"
  '';
}
```

Which could be pretty cool:

```sh
sam@vcs ~> nix-shell test.nix
using python: python3-3.6.4

[nix-shell:~]$
```


However, we couldn't use the `py` variable in `shellHook` if the let-in expression only covers the `buildInputs` list.

```nix
# THIS CODE WILL CRASH
with import <nixpkgs> {};

stdenv.mkDerivation rec {
  name = "python-environment";

  buildInputs = let
    py = pkgs.python36;
  in [
    # `py` is now in scope
    py
    py.pkgs.flask
    py.pkgs.itsdangerous
    py.pkgs.six
  ];
  # `py` is now out of scope (a list is one type of "expression")

  shellHook = ''
    echo "using python: ${py.name}"
  '';
}
```

It would result in a crash:

```
> nix-shell test.nix
error: undefined variable ‘py’ at /home/sam/test.nix:20:27
```

## Extension: the "with" expression

With is another expression (like let-in).  It has the syntax:

```nix
with expression1; expression2
```

With actually works very similar to the JavaScript with, and nothing like the python with.  Basically, it:

1.  It evaluates expression1; call that `ret1`. `ret1` must be a set (aka. dictionary)
2.  It takes all the attributes of `ret1`, and makes them variables in the scope of expression2
3.  Overall, it evaluates to expression2 (which run with the extra variables)

So you could replace:

```nix
let
  x = 1;
  y = 2;
in
  x + y
```

With the `with` equivalent:

```nix
with { x = 1; y = 2; }; x + y
```

This is really useful when dealing with sets that have loads of attributes, like `python36.pkgs`.  So our old code:

```nix
buildInputs = [
  pkgs.python36
  pkgs.python36.pkgs.flask
  pkgs.python36.pkgs.itsdangerous
  pkgs.python36.pkgs.six
];
```

Could become shorter using `with`:

```nix
buildInputs = with pkgs.python36.pkgs [
  pkgs.python36
  flask
  itsdangerous
  six
];
```

As you can see, all the attributes of `pkgs.python36.pkgs` (including `flask`, `itsdangerous` and `six`) were added to the scope when evaluating the list.

This can also be chained with the let-in expression:

```nix
buildInputs =
  let
    py = pkgs.python36;
  in
    with py.pkgs;
    [
      py
      flask
      itsdangerous
      six
    ];
```

## Up Next

Creating a super simple derivation - Learning Nix pt 3

Follow the series [on GitHub](https://github.com/samdroid-apps/nix-articles)

*Hero image from [nix-artwork by Luca Bruno](https://github.com/NixOS/nixos-artwork/blob/master/gnome/Gnome_Dark.png)*
