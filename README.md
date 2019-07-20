# newrepo-findrepo - tools for small-scale Git servers

`newrepo` and `findrepo` are utilities that allow users with shell access to
manage Git repositories. They are intended primarily for remote use via SSH,
though you can also use them locally. They run on Unix-like systems, but there
is of course no limit on what system you SSH in from. The user running them
should have access to traverse into and read the directory where repositories
are kept, and needs write access to create new repository.

I use these tools in production. However, for general use, I suggest
considering them beta-quality (at best). They're missing some desirable
features and haven't been widely tested. As far as I'm aware, they're only
deployed long-term on one machine in the world.

See below for usage.

### Configuration

Currently, the path `/repos` is hard-coded as the directory where repositories
are kept. This can be modified in `findrepo` and `newrepo`. **The only external
configuration is the domain name (or IP address) that will be used in the
`ssh://` URLs the tools output.** Using the ouptut of `hostname`, or other such
methods, would not be reliable, especially when the server is a machine in a
home or small office connected to the Internet with a NAT router, which is the
primary use case for these tools.

`newrepo` and `findrepo` look for `/etc/newrepo-findrepo.conf` and expect it to
contain a line of the form

```none
host=HOST
```

where `HOST` is a domain name or IP address.

For example, if that file consists of the line `host=example.com`, then an
argument of `gnomovision` passed to `newrepo` or `findrepo` refers to a
repository intended for access by the remote URL
`ssh://example.com/repos/gnomovision.git`.

### Setting up your repositories directory

I suggest making a group, e.g., `coders`, membership in which confers the
ability to write to `/repos`. Make `/repos` a setgid directory with `root` as
its owner and `coders` (or your chosen group name) as its group owner.

For example, on Debian with coders `mary` and `larry`:

```sh
sudo addgroup coders
sudo mkdir /repos
sudo chgrp coders /repos
sudo chmod g+ws,o-rwx /repos
sudo adduser mary coders
sudo adduser larry coders
```

This setup assumes all members of `coders` are trusted with access to all
repositories. In particular, it permits users in that group to modify and even
delete their fellow users' repositories.

### SSHing from any system (but especially Unix-like systems)

One way to use these utilities from client machines is to run commands like:

```sh
ssh example.com newrepo gnomovision
```

```sh
ssh example.com findrepo gnomovision
```

Of course, you may wish to automate this with a script, shell function, or the
like. Automating them on Windows can be slightly more complicated, so...

### SSHing from Windows (supporting custom `%GIT_SSH%`, e.g., with `plink`)

Convenient key-based SSH authentication can be a bit challenging on Windows,
so `nrr` and `frr` commands are provided in the separate **nrr-frr** repository
to help Windows clients run `newrepo` and `findrepo` on a server.

This isn't actually necessary, but it can make it easier to authenticate the
same way you when peforming remote actions with Git. For example, if you have a
custom `%GIT_SSH%` environment variable that (directly or indirectly) runs
`plink`, you might find nrr-frr helpful.

### Using `newrepo`

`newrepo` creates a new bare Git repository in a top-level repo directory and
reports the URL that can be used to clone it or add it as a remote. The name of
the repository to be created is passed as a command-line argument.

```sh
newrepo NAME
```

It looks like this when it succeeds at creating a repository:

```none
$ newrepo gnomovision
Initialized empty Git repository in /repos/gnomovision.git/

The URL of the newly created repository is:

        ssh://example.com/repos/gnomovision.git
```

When it fails, its output is unlikely to be misunderstood as success:

```none
$ newrepo gnomovision
mkdir: cannot create directory ‘gnomovision.git’: File exists
```



### Using `findrepo`

`findrepo` searches repositories in a top-level repo directory. It assumes the
users intends an exact match but might be wrong, perhaps due to typos, but as
likely to incomplete or mistaken memory about the name of the reposory they're
looking for. The search pattern is passed as a command-line argument.

```sh
findrepo GUESS
```

Your guess can be pretty wildly wrong, and `findrepo` still usually manages to
include what you meant in its small handful for suggestions.

`findrepo` shows a URL when it's pretty sure that specific URL is what you
actually want (for cloning a repo or adding a remote). When there is any
substantial ambiguity, is shows its own guesses but not a URL. You can run it
again with one of those guesses to get a URL.

Even when there is no real ambiguity, it warns about situations like incorrect
capitalization that might cause confusion under some circumstances.

### How `findrepo` behaves

When your guess is correct, it gives you the URL for the repository:

```none
$ findrepo gnomovision
The URL for the repository "gnomovision" is:

        ssh://example.com/repos/gnomovision.git
```

When there are other repositories that would have matched except for
differences in capitalization (a situation that would typically arise only by
accident), it gives yout the URL but also warns:

```none
$ findrepo TestRepo
The URL for the repository "TestRepo" is:

        ssh://example.com/repos/TestRepo.git

BEWARE! 2 other repos are named like this but capitalized differently.

        TeStRePo
      * TestRepo
        testrepo
```

When your guess is off only by case, `findrepo` responds appropriately
depending on whether or not there is ambiguity:

```none
I have no "GnomoVision" repo. Perhaps you mean "gnomovision"?

The URL for the repository "gnomovision" is:

        ssh://example.com/repos/gnomovision.git

BEWARE! The filesystem on this server is case-sensitive.
```

```none
$ findrepo TESTREPO
I have no "TESTREPO" repo. Maybe you want one of these...

        TeStRePo
        TestRepo
        testrepo

Those repos are like what you requested, but with different capitalization.
If any of those look EXACTLY the same, they may have weird Unicode characters.
```

The more interesting situation is when you spell the repository name
incorrectly, or just guess wrong:

```none
$ findrepo gnomviissn
I have no "gnomviissn" repo. These repos have vaguely similar names...

        gnomovision
        NVI
        Visit
        AsIs
        Disposing
```

```none
$ findrepo visual-moon
I have no "visual-moon" repo. These repos have vaguely similar names...

        gnomovision
        foo
        Pool
        Visit
        Mob
        NVI
```

### How `findrepo` works, under the hood
