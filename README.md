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
Still, it relies on some nixos-related web services, similar to docker and hub.docker.org. For example, cache.mixos.org.

## able to handle baremetal OS

Docker/Ansible/Chef/Puppet designed to run as an OS service. NixOS is designed to handle all the aspects of OS: starting from handling boot loader to containers, display managers and PCI passthrough to virtual machines.

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

First of all, to use NixOS you need to learn to some degree the Nix expression language, which is not difficult if you already familiar with functional languages, but still is not being used outside NixOS/nix-package-manager. There is another point of view, in which "learn the nix expression language and forget a time, when you needed to understand a bunch of syntax of bunch of different programs" exist as well.

### Writing nix-expressions for programs

Although there are many programs, that already have their nix-derivations and many NixOS services(modules), it is still not the standard for development and there will probably be a case when you will need to create your own nix-derivations and NixOS modules yourself. It is not hard, but still require additional time to be spent in terms of 'user experience'. It requires some experience with NixOS tooling/library (but not a huge experience, tough).

## Patching programs to fit nix flow

`nix` package manager stores nix-derivation in the read-only `/nix/store` directory. `nix` relies on a hash of the nix-derivation and any change to it will change the hash and trigger rebuild. This also means, that programs, that do expect to have some config in the program's dir will trigger a derivation to be rebuilt everytime this config will be changed. Mempool backend is the example of such application.

Ie, the canonical way is to keep configuration file as a separate nix-derivation, which change will not trigger application to be rebuilt: just services and config file will be rebuilt and will trigger service to restart.

## There is no checks of the bash scripts

You still can have a misbehaved service, that can mess the state, for example, which can remove the content of /etc or stop systemd services.

At the same time, you can use `nixos-containers` to run services in an isolated environment in a similar way as docker or third-party tools as a hooks to check the bash scripts.

## NixOS tools can't be called a 'lightweight'

I saw, that `nixos-rebuild switch` command can occupy 2+ GiB of RAM and the running time can be far from instant. I haven't checked the sources to get why, but I can GUESS, that, as in Haskell: lazy evaluation is not comes for free and delaying computation of something, means, that you need to keep the arguments of such computation in memory and rely on a good garbage collector.

# Comparison to some alternatives

Based on solving issues, I will compare NixOS with those alternatives:
- docker/kubernates;
- chief;
- puppet;
- ansible.

First of all, I will comment Conses of NixOS in terms of other tools.

## Cons

### Nix expression language

Any of NixOS alternatives require to know how to actually do anything: either they are named "recipes" or "Docker file" and so on. So the requirement to learn `Nix` expression language is not that different of the requirement of knowning how to write recipes or Docker files for the alternatives.

#### Writing nix-expressions for programs

NixOS alternatives require to define interaction of services, but usually rely on the host OS's package management and so usually there is no appropriate issue with the alternatives.

### Patching programs to fit nix flow

Alternatives usually do not require to modify programs to fit their flow as they usually trying to not to add constraints on the existing eco system of the host OS.

### There is no checks of the bash scripts

The same issue exist in the alternatives.

## NixOS tools can't be called a 'lightweight'

Ansible/Puppet/Chef are relying at least on database server and application server, which I doubt can be a `ligther` alternative to NixOS tools, that are being used during `nixos-build` execution.

Docker can be considered as a lighter alternative. In terms of RAM usage.

## Pros

Now, I will compare NixOS proses with alternatives.


## does not require any additional services to maintain

Ansible/Puppet/Chef/kubernates are designed to work as a centralized network services with appropriate agents running on the hosts. So users of such alternatives have to maintain appropriate infrastructure.

## able to handle baremetal OS

NixOS alternatives such as Ansible/Chef/Puppet are usually able to handle whole OS configuration, but they are usually respect more network-related parts of network management.
Docker is designed to be used as a service of some Host OS. The closest case to handling whole OS I am aware of is Balena and it's 'balena hypervisor' OS, which is basically a custom linux distro, which goal is to be used as host, which connects to Balena platform and to be controlled from the Balena cloud.

## able to switch between OS configurations on the fly or from grub menu during boot

Ansible/Puppet/Chef do not track for "previous" configurations, so there is no a "reverting" in the case of switching to a configuration. Instead, they provide a way to catch errors and perform some actions to react on errors, such that they can undo changes, that fail assertations, which may or may not be viewed as a way of reverting configurations.

At the same time, I can't say, that reverting configuration is an oftenly used feature: I had to use it like 3-4 times, but I like, that I had such ability.

## be able to provide reproducible environment

Ansible/Puppet/Chef are designed to manage host OS, which is usually operate on the shared state, it is much harder to get fully reproducible environment with them in compare to NixOS as order of installation affects the result in most such host OSes.
Docker is commonly using the same OSes with shared environment, so the order of installation does affect the result as well.

## able to handle dll-hell issue

Ansible/Puppet/Chef rely on the host OS's package manager, so they are offloading the solution of this issue to a host OS, which are usually bad at this.
One of the goals of Docker is to isolate services from each other, which may be used as a solution for dll-hell issue, but, in my opinion, this is more a side-feature of the isolation than the completed solution, as you can still have dependencies issues within one container.

## able to perform syntax / assertions checks during config switch

NixOS alternatives do have support of checking for syntax errors, although they are usualyl rely on a yaml/json-like formats, that are far from being domain-specific.

## More granular control

Docker is using "layers", which are implemented as a multi-layered overlayfs mount points, to track the changes and reuse layers, that haven't been changed. Although, this works for isolating filesystem changes, it still require to restart the whole container during update of one service.
Ansible/Puppet/Chef are able to restart only affected services during update, though, they are relying on the host OS's package manager and I am not sure how easy to crash them in case of "poweroff during update" (ie, for example, when `apt` asks to run `dpkg --configure -a`). NixOS builds the config first, before actual switching to it, so NixOS will not be in a broken state in case of poweroff during update.

## Can be used as a Docker host

Docker is able to run a nested Docker service.
Ansible/Puppet/Chef can manage docker service as well.

Non of them can run/manage NixOS. Docker can only run a container with `nix` package manager, but not the whole NixOS.

So NixOS is more universal in the sense, that it can be used to manage all the alternatives and serves as a host for `nixos-containers` or for Docker service.

# Conclusion

First of all, this comparison is very opinionated.
There are many tools, that solves the same issues, which NixOS is targetting to solve and any of those will shine in specific use cases.
But, in my opinion, NixOS is a good tool, which solves number of issues effectively and does not forces you to maintain network infrastructure to use it.
