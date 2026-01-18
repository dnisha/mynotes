---
longform:
  format: single
  title: Environment & Shell Configuration
title: Environment & Shell Configuration
---
## **Environment Variables**

```bash

# View variables
env                       # All environment variables
printenv                  # Same as env
echo $PATH                # Specific variable
set                       # Shell variables + environment

# Set variables
export VARNAME="value"    # Temporary (current session)
# Permanent: add to ~/.bashrc or ~/.profile

# Common variables
PATH                      # Command search path
HOME                      # Home directory
USER                      # Username
SHELL                     # Current shell
PS1                       # Prompt string

```

## **Common configuration**:

```bash

# ~/.bashrc - Interactive shell config
alias ll='ls -la'
alias ..='cd ..'
export EDITOR=vim
export HISTSIZE=10000
export HISTFILESIZE=20000

# Reload config
source ~/.bashrc
. ~/.bashrc

```