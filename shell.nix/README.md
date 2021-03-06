
# shell.nix

[The nix haskell tutorial](https://github.com/Gabriel439/haskell-nix/tree/master/project0#building-with-cabal) explains how to use Nix to provision a shell environment,
in which you get all the tools required to build your package with Cabal.

Cabal newly features a new v2 API (currently exposed with the `new-` prefix) that allows you to build multi-package projects.
This is ideal for monorepos.

Ideally we would like to combine the above two mechanisms and have Nix provision an environment in which we would have a GHC with all
the dependencies of all our packages, such that the only task left to Cabal is actually building whatever we are working on.

> NOTE
>
> Cabal used to have an `--offline` flag that would assure it couldn't fetch any packages from the internet.
> It seems it [no longer works with the new API](https://github.com/haskell/cabal/issues/5346).
> If you really want to make sure Cabal is using the packages provided by Nix you can use the following hack ([source](https://github.com/haskell/cabal/issues/5783#issuecomment-445518839)):
>
> 1. Remove all the files and directories except `config` in the `~/.cabal` folder, so that any already fetched packages will be removed
> 2. Remove any `dist` folders that were results of previous `cabal` runs
> 3. Comment out the repository section in your `~/.cabal/config` like so:
> ```
> -- repository hackage.haskell.org
> --   url: http://hackage.haskell.org/
> --   secure: True
> --   root-keys:
> --   key-threshold: 3
> ```
>
> Now Cabal should really have no way to access packages other than the ones already installed with GHC.

## Cabal `new-*` commands

To make use of the new multi-package support, you have to declare a `cabal.project` file. In the trivial case
you will simply list the constituent packages like so:

```
packages: hello-world
          universe
```

If you now run `cabal new-configure && cabal new-build all` Cabal will first fetch all the dependencies of both `hello-world` and `universe` and it will then
build the packages in the correct order until everything is built. Instead of "all" you can also specify a single package and Cabal will only build the
very minimum to build that one package.

Which versions of the dependencies will Cabal fetch is a complicated topic and can be entirely avoided by using Nix.

> NOTE
>
> If you were already using Cabal to build your monorepo and you are specifying external dependencies explicitly,
> you will want to create a second `cabal.project` in the nix folder without those external deps.
> Then you will have to specify the `caba.project` file with all the cabal invocations. e.g.
> ```
> cabal new-configure --project-file ../nix/cabal.project && cabal new-build --project-file ../nix/cabal.project
> ```
> It is unfortunate that Cabal doesn't allow to specify the project file with an environment variable instead of a flag,
> since we can specify environment variables in shell.nix files.

# Nix as package provisioner

Every haskell derivation in Nix provides us with a `env` attribute.
If we try to create a shell with this attribute (e.g. `nix-shell -A haskellPackages.aeson.env "<nixpkgs>"`) we will end up with a GHC that has
all the dependencies of the given package already installed. When we subsequently run Cabal it will realize that there is no fetching to be done
and proceed to only build the requested package.

What we would like though in our case, is a the environment for a __set__ of packages.
All the dependencies are listed in the arguments of the Nix expressions we generated for our Haskell packages.
We could create a program that would go over the nix expressions in `packages`, parse the arguments and output the concatenated list of them into a dummy Haskell package Nix expression.
This is exactly what I tried at first and [here](https://github.com/fghibellini/nix-scripts/tree/master/monorepo-gen-env) you can find the source for it.

I then realised the expression can be generated dynamically by using the Nix builtin functions.

> NOTE
>
> After having implemented the below described solution I realised it should be possible to do the same,
> but in a more reliable manner with less code by using `<haskellPackge>.getBuildInputs.haskellBuildInputs`.
> I described this in a [separate file](./APPROACH2.md) as it currently doesn't seem to work properly.

We start off by creating the template for the expression:

```nix
# monorepo.nix
let
  nixpkgs = import ./release.nix;
  all-deps = []; # list of string package names - this is what we need to figure out how to generate
in
  nixpkgs.haskellPackages.mkDerivation {
    pname = "monorepo";
    version = "1.0.0";
    src = null;
    libraryHaskellDepends = map (pkgName: builtins.getAttr pkgName nixpkgs.haskellPackages) all-deps;
    license = nixpkgs.stdenv.lib.licenses.unfree;
  }
```

To actually compute the list of all deps we will use a hacked up function:

```nix
{ lib }:
let
    unique = lib.lists.unique;
in
    dir: let files = builtins.readDir dir;
             project-names = builtins.attrValues (builtins.mapAttrs (name: value: builtins.elemAt (builtins.match "(.*)\\.nix" name) 0) files);
             args = builtins.mapAttrs (name: value: builtins.attrNames (builtins.functionArgs (import (dir + "/${name}")))) files;
             union = unique (builtins.concatLists (builtins.attrValues args));
        in
            builtins.filter (pkg: ! builtins.elem pkg (["stdenv" "mkDerivation"] ++ project-names)) union

```

The function code looks very cryptic, but it really does what we want. You can simply run the function on our `packages` folder and see it indeed returns a list of strings representing the dependencies.

```bash
$ cd nix
$ nix-instantiate --eval -E "((import <nixpkgs> {}).callPackage ./lib/compute-monorepo-deps.nix {}) ./packages"
[ "aeson" "base" "text" ]
```

Now we can simply replace the placeholder empty list with the function call:

```
# monorepo.nix
...
  nixpkgs = import ./release.nix;
  all-deps = (nixpkgs.callPackage ./lib/compute-monorepo-deps.nix {}) ./packages;
in
...
```

This dummy package now allows us to create the desired shell environment:

```bash
nix-shell -A env ./monorepo.nix
```

Furthermore `nix-shell` will by default try to read first a `shell.nix` file in the current directory to generate a shell.
We can greatly take advantage of this and add a `shell.nix` file to our `code` folder with the following contents:

```nix
# shell.nix
(import ../nix/monorepo.nix).env
```

Now any user can simply move to the folder containing our haskell packages and run `nix-shell` without any arguments and he will be
thrown in a shell with all the deps required to run a simple `cabal new-configure && cabal new-build all`.

```bash
$ cd code
$ nix-shell
[nix-shell:.../monorepo/code]$ cabal new-configure
Warning: No remote package servers have been specified. Usually you would have
one specified in the config file.
Resolving dependencies...
Build profile: -w ghc-8.4.4 -O1
In order, the following would be built (use -v for more details):
 - universe-0.2.0.0 (lib) (first run)
 - hello-world-0.1.0.0 (exe:hello-world-exe) (first run)

[nix-shell:.../monorepo/code]$ cabal new-build all
Build profile: -w ghc-8.4.4 -O1
In order, the following will be built (use -v for more details):
 - universe-0.2.0.0 (lib) (first run)
 - hello-world-0.1.0.0 (exe:hello-world-exe) (first run)
Configuring library for universe-0.2.0.0..
clang-5.0: warning: argument unused during compilation: '-nopie' [-Wunused-command-line-argument]
Preprocessing library for universe-0.2.0.0..
Building library for universe-0.2.0.0..
[1 of 1] Compiling Universe.World   ( src/Universe/World.hs, .../monorepo/code/dist-newstyle/build/x86_64-osx/ghc-8.4.4/universe-0.2.0.0/build/Universe/World.o )
Configuring executable 'hello-world-exe' for hello-world-0.1.0.0..
clang-5.0: warning: argument unused during compilation: '-nopie' [-Wunused-command-line-argument]
Preprocessing executable 'hello-world-exe' for hello-world-0.1.0.0..
Building executable 'hello-world-exe' for hello-world-0.1.0.0..
[1 of 1] Compiling Main             ( exe/Main.hs, .../monorepo/code/dist-newstyle/build/x86_64-osx/ghc-8.4.4/hello-world-0.1.0.0/x/hello-world-exe/build/hello-world-exe/hello-world-exe-tmp/Main.o )
Linking .../monorepo/code/dist-newstyle/build/x86_64-osx/ghc-8.4.4/hello-world-0.1.0.0/x/hello-world-exe/build/hello-world-exe/hello-world-exe ...
clang-5.0: warning: argument unused during compilation: '-nopie' [-Wunused-command-line-argument]
clang-5.0: warning: argument unused during compilation: '-nopie' [-Wunused-command-line-argument]

[nix-shell:.../monorepo/code]$ cabal new-run hello-world-exe
Up to date
Hello WORLD!!!
```

You can also add the following to your `.gitlab-ci.yml` if you want to make sure the above command trio will always work:

```
nix-shell:
  stage: build-and-lint
  image: fghibellini/nix # https://github.com/fghibellini/nix-haskell-gitlab-runner
  allow_failure: true
  script:
    cd $CI_PROJECT_DIR/code && nix-shell --command 'cabal new-configure && cabal new-build all'
```

In the [next chapter](../setting-up-a-hydra-instance) we will see how to get up and running a NixOS machine with a Hydra daemon.

