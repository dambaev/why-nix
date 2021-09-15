# Brief

This document describes what issues solves NixOS and which way in comparison to other tools.

# Issues

The main issue, that NixOS is trying to solve are:
1. providing reproducible environments;
2. providing consistent(non-conflicting) environment for projects, that require different runtimes without isolating them;
3. handling 'dll-hell' case.

# How does it solves those issues

  NixOS relies on `nix` as a package manager, that is using `Nix` purely functional language.
  `nix` as a package manager relies on a builds, that are running in an isolated (in terms of filesystem and network) environment and tries to provide as clean environment as possible. Such isolated environment will only contain an explicitly defined dependencies. This solves the case when the developer has some libraries installed on his OS in the past, but he does not know that those libraries are being used by the program. The result of such build is being called `nix derivations`. Each `nix-derivation` is being stored in the `/nix/store` path with hash of the source of the build, such that:
1. `nix` package manager defines a functional dependency on a build inputs, build declaration and the hash of the result;
2. `nix` package manager will not perform any rebuild if the calculated hash of the `nix-derivation` is already contained in the `/nix/store`;
3. different versions of the same library will have different path in `/nix/store`, allowing to use such different versions in different programs (ie, solving `dll-hell` issue).

  `Nix` expression language has builtin type 'Set', which represents a set of named values, which are of a primitive type or set themselves. NixOS extends the library of `Nix` expression language to represent the whole OS configuration as a single set of options. By having an operator which composes 2 sets in 1, NixOS allowes to define multiple nix-expressions, which define a set of NixOS options. And by composing them, NixOS gets the resulting configuration of the OS as a single set.
  Although, NixOS still allowes you to use `sysctl` / `sysfs` and so on to set a temporary changes to your system, the recommended way is to declare your OS configuration in `/etc/nixos/configuration.nix`, which is a way to make such changes to be a persistent across reboots / deployments.
  Keeping the whole OS configuration in one file allows to get a configuration management system and to have a reproducible environment. Another nice feature of such way of keeping the Os configuration is an ease of storing it in the version control system, such as git.
  
# Pros

## does not require any additional services to maintain

NixOS is self-hosted, unlike Ansible, Puppet and etc it does not require any additional service to run in the network to work.
Still, it relies on some nixos-related web services, similar to docker and hub.docker.org.

## able to handle baremetal OS

Docker/Ansible/Chef/Puppet designed to run as an OS service. NixOS is designed to handle all the aspects of OS.

## able to switch between OS configurations on the fly or from grub menu during boot

I had a case when I had pushed a wrong network config config to one of the servers. In this case it was just enough to reboot the server and choose the previous OS configuration from the grub boot menu.
Similar feature can be achieved with any linux + ZFS snapshots, but ZFS snapshots are less granular, as it works only on a filesystem level: ZFS had to snapshot /etc, /bin, /sbin, /usr/, some dirs of /var (but not all, because /var/lib can contain data).
Switching ZFS snapshots online will not trigger services to be restarted, so it should be done offline. In NixOS it is possible to revert configuration online: it will restart only the necessary services affected by such revert.
Ie, NixOS configuration are more granular and advanced in this sense.

## be able to provide reproducible environment

On majority OSes each previously installed program may affect the build of the other programs in the future, such that you may not know what the actual dependencies of the program are, such that changing the order of installation may affect the result. NixOS builds nix-derivations in an isolated environment, such that all the dependencies should be expclicitly listed for such derivations. This means, that:
1. nix-derivations do not rely on the order of installation of other programs;
2. it is much more harder to get different environment for the same NixOS configuration (I see the only case if you get different result for the same config in different OS releases).

## able to handle dll-hell issue

This case had been mentioned in the case above, but I think, that it deserves separate case as well: NixOS is able to handle the case when different programs require the same library but of different versions without using namespaces (containers).9

## able to perform syntax / assertions checks during config switch

before actual switching to the new configuration NixOS checks nix-expressions/nix-derivations for syntax errors. Those expressions can contain assertions checks, which will be checked as well. Failure at any of those checks will prevent NixOS to switch to the incorrect configurations and so: trying to apply incorrect configuration will not affect any running services.

## More granular control

Addition / removing of service affects only this service. Ie, NixOS does not 'restarts the whole container' as docker does. NixOS does not rely on containers at all. Instead it is about 'I have new configuration: I have to stop unused services and to start new services'

## Can be used as a Docker host

NixOS can run docker service to have a mixed environment

# Cons

## Nix expression language

First of all, to use NixOS you need to learn to some degree the Nix expression language, which is not difficult if you already familiar with functional languages, but still is not being used outside NixOS/nix-package-manager.

### Writing nix-expressions for programs

Although there are many programs, that already have their nix-derivations and many NixOS services(modules), it is still not the standard for development and there will probably be a case when you will need to create your own nix-derivations and NixOS modules yourself. It is not hard, but still require additional time to be spent in terms of 'user experience'. It requires some experience with NixOS tooling/library (but not a huge experience, tough).

## Patching programs to fit nix flow

`nix` package manager stores nix-derivation in the read-only `/nix/store` directory. `nix` relies on a hash of the nix-derivation and any change to it will change the hash and trigger rebuild. This also means, that programs, that do expect to have some config in the program's dir will trigger a derivation to be rebuilt everytime this config will be changed. Mempool backend is the example of such application.

Ie, the canonical way is to keep configuration file as a separate nix-derivation, which change will not trigger application to be rebuilt: just services and config file will be rebuilt and will trigger service to restart.

## There is no checks of the bash scripts

You still can have a misbehaved service, that can mess the state, for example, which can remove the content of /etc or stop systemd services.

At the same time, you can use `nixos-containers` to run services in an isolated environment in a similar way as docker or third-party tools as a hooks to check the bash scripts.

# Comparison to some alternatives

Tools like docker/kubernates/chief/puppet/ansible do require to write 'recipes'/service definitions as well, but usually do not require patching (as they just reusing the host OS package management).
