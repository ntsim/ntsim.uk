---
id: 'b9c87af1-3fe9-4e37-9597-aaa081f5b5b4'
title: 'Running Git hooks with environment variables in IntelliJ'
description: 'How to run Git hooks (with environment variables) in IntelliJ without losing your mind. Perfect for NVM or SDKMAN users.'
date: '2020-12-28 17:51:00'
tags:
  - IntelliJ
  - Git
  - Linux
---

If you use the IntelliJ (or any JetBrains IDE) commit GUI in combination with the following:

- A language version manager like [NVM](https://github.com/nvm-sh/nvm) or [SDKMAN](https://sdkman.io/) (you should)
- Git hooks (e.g. via [Husky](https://github.com/typicode/husky)) that rely on some environment variable like `PATH` or `JAVA_HOME`

You'll most likely have encountered errors like this when you try to commit:

> JAVA_HOME is not set and no 'java' command could be found in your PATH. Please set the JAVA_HOME variable in your environment to match the location of your Java installation.

Obviously, this is unlikely to be the case as you can run Java, Node or whatever through your terminal. So what's the problem?

## The problem

If you open up IntelliJ using the terminal (using the `bin/idea.sh` script), you'll find that your Git hooks will suddenly start working with the commit GUI. Obviously, this isn't great as you'll usually want to open IntelliJ up via a desktop launcher for convenience, and to avoid keeping an extra shell open.

Language version managers like NVM or SDKMAN work by hooking into `.bashrc` or `.zshrc` startup scripts. If you look inside one of these, you'll see something like:

```
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

Unfortunately, opening IntelliJ through a desktop launcher doesn't run `.*rc` startup scripts like in a terminal. This is because desktop applications are opened in _non-interactive_ shells (unlike terminal shells, which are _interactive_). Launching IntelliJ in this way will not execute any of the commands needed by the language version managers.

## The solution

There's a few solutions to this problem, but the one I find the simplest is to add any required environment variables to `.profile`, `.bash_profile` or `.zprofile` in your home directory. These startup scripts are ran by your desktop shell upon login and anything you define in here will be seen correctly by IntelliJ.

I prefer to use `.profile` for simplicity and have something like this in it:

```
export PATH="$HOME/.nvm/current/bin:$PATH"
export JAVA_HOME="$HOME/.sdkman/candidates/java/current"
```

Note that in the above, the 'current' directories are just symlinks to the actual locations.

Finally, just log in again and you should see that your Git hooks now work with the commit GUI. Nice and easy.
