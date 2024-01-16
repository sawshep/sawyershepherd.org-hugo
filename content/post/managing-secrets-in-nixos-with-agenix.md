---
title: "Managing Secrets in NixOS With Agenix"
date: 2024-01-15T16:49:21-05:00
draft: false
---

# Introduction

NixOS offers great declarative system configuration, but by
design, every output is stored in the world-readable `/nix/`
directory. Sooner or later, you'll find that you need to
configure an option with secrets that you don't want
world-readable, let alone in a public Git repository!

[Agenix](https://github.com/ryantm/agenix) solves this issue
by encrypting secrets with host SSH keys via the
[Age](https://github.com/FiloSottile/age) application and
decrypting them at build time.

I found that the documentation of Agenix is a little lacking
in clarity, so I thought I'd review how I implement and use
Agenix. Regardless, the documentation is a good resource if
you want to dive deeper into Agenix. This guide assumes that
you restrict switching builds to superusers, and that you do
not want to permit regular users from decrypting system
secrets.

# Installation

The easiest way to install Agenix into your config is with
the `fetchTarball` builtin in your main configuration. I
hate managing channels and using `fetchTarball` prevents
that.

```nixos
{ config, pkgs, ... }:
{
  imports = [
    "${builtins.fetchTarball "https://github.com/ryantm/agenix/archive/main.tar.gz"}/modules/age.nix
    ./my-other-import.nix
    ...
}
```

Additionally, we will have to have a host key. The easiest
way to generate a host key is to set `system.ssh.enable =
true;` in your configuration. Your host key will be created
at `/etc/ssh/ssh_host_*_key`. This key will be used to
automatically decrypt secrets when building a new
generation. I like the short length of ED25519 keys, so I
will be using those.

If you don't wish to enable SSH, you can also manually
generate a host key and configure Agenix to use it. It's a
bit of a pain so I'm not going to cover it here, but you can
find documentation on how to do so in the [module
reference](https://github.com/ryantm/agenix#age-module-reference).

# Declaring Secrets

With Agenix installed, we can create the
`/etc/nixos/secrets` directory. In this directory, we'll
declare our actual secrets via the `agenix` binary, and then
a list of keys that we want to be able to decrypt them in
`secrets.nix`. We must first declare our keys in
`secrets.nix` so Agenix will know what recipients to encrypt
our secrets with once we get to creating them. This is what
my secrets file looks like:

```nixos
let
  users = import ../authorized_keys;

  laptop =      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPsLlqDXl8kU+FfzJQpzPXQCfJdntfEDIaSDyfezy5Hy root@elitebook-835-g7";
  server =      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEJEBV9qA0qZIBdu1aL8DjXpKdtzl+Pf48LAy8PUaY3Q root@changwang-cw56-58";
  workstation = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAII36AUr8me43Oj6ZTqgG+hylGl9jwny6m1wTtZERoxUo root@asustek";
  seedbox =     "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIbmZKe4en/4xIyxuL3DrE4W/HUmE61lbBXNs/HUik3Y root@seedbox";

  systems = [ laptop server workstation seedbox ];
in
{
  "user-password.age".publicKeys = systems;
  "caddy-basicauth.age".publicKeys = [ server ];
}
```

In the `let` block, I import a list of my users' SSH keys
from another file to make my configuration more modular as
those keys are also used elsewhere in my configuration. I
also declare the public **host key** of each of my systems,
found at `/etc/ssh/ssh_host_*_key.pub` It's not the SSH key
of the root user!

In the main block, I declare my secret files and what keys
will be able to decrypt them. You must declare your secrets
before creating them. The recipient public keys must be
provided as a list. Typically, if a secret is being used on
a host, you want to list that host as a recipient so it can
be decrypted at build time.

Let's declare a new secret `my-secret.age` that we want both
all users and all systems to be able to decrypt. First, we
declare the secret in `secrets.nix`:

```nixos
let
  users = import ../authorized_keys;

  laptop =      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPsLlqDXl8kU+FfzJQpzPXQCfJdntfEDIaSDyfezy5Hy root@elitebook-835-g7";
  server =      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEJEBV9qA0qZIBdu1aL8DjXpKdtzl+Pf48LAy8PUaY3Q root@changwang-cw56-58";
  workstation = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAII36AUr8me43Oj6ZTqgG+hylGl9jwny6m1wTtZERoxUo root@asustek";
  seedbox =     "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIbmZKe4en/4xIyxuL3DrE4W/HUmE61lbBXNs/HUik3Y root@seedbox";

  systems = [ laptop server workstation seedbox ];
in
{
  "user-password.age".publicKeys = systems;
  "caddy-basicauth.age".publicKeys = [ server ];
  "my-secret.age".publicKeys = systems ++ users;
}
```

# Creating Secrets

With our secret declared in `secrets.nix`, we can actually
create our secret. While in `/etc/nixos/secrets`, run
`agenix -e my-secret.age`. Your `$EDITOR` will open and you
can enter your data you wish to keep secret. Save and quit,
and the secret will automatically be encrypted to the
recipient keys you specified in `secrets.nix`.

We can decrypt our secrets with either one of our host keys
or one of our user keys:

```bash
/etc/nixos/secrets/ $ agenix -d my-secret.age -i /etc/ssh/ssh_host_ed25519_key
/etc/nixos/secrets/ $ agenix -d my-secret.age -i ~/.ssh/id_ed25519
```

Additionally, if we declare more recipients for our secret
in `secrets.nix`, we can rekey the secret with:

```
/etc/nixos/secrets $ agenix -r my-secret.age -i /etc/ssh/ssh_host_ed25519_key
```

Note the secret file will always be different after rekeying
it, regardless of any more recipients are added.

# Using Secrets

You can use your secrets by defining them under the
`age.secrets` submodule, then by calling their path as shown
below:

```nixos
{
  age.secrets.my-secret.file = ./secrets/my-secret.age
  users.users.myuser = {
    isNormalUser = true;
    passwordFile = config.age.secrets.my-secret.path;
  };
}
```

Agenix only exposes the file path to the decrypted secret,
as exposing the secret itself would make it world-readable.

Good luck, and remember as Gandalf said, "Keep it secret.
Keep it safe!"
