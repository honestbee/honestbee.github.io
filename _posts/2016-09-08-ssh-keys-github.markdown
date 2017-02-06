---
layout: post
title:  "A clean process to get ssh keys from Github"
excerpt: "Recently we decided to improve the way we manage ssh keys on our servers. We’re running our stack on Elastic Beanstalk and the way we would push our ssh keys would be to use a config file appending the user’s key to the server’s authorized_keys file. This solution is not really optimal because it means we’re storing the public keys in the repository, access control is quite low and you’d have to go through a whole pull request process just to be able to ssh to a machine."
date:   2016-09-08 12:13:33 +0800
categories: [devops]
tags: [devops,aws]
author: Charles Martinot, Vincent De Smet
---
Recently we decided to improve the way we manage ssh keys on our servers. We're running our stack on Elastic Beanstalk and the way we would push our ssh keys would be to use a config file appending the user's key to the server's `authorized_keys` file. This solution is not really optimal because it means we're storing the public keys in the repository, access control is quite low and you'd have to go through a whole pull request process just to be able to ssh to a machine.

# Goals
1. Ideally, we'd like to be able to organise users in groups, each groups having access to different servers.
2. Teams would manage who has access themselves.
3. Revoking/Granting access should be immediate.
4. Public keys wouldn't be stored in a repository.
5. Users should be able to access the servers from different machines.

# Solution
A colleague suggested [this solution][better-ssh]. Using the `AuthorizedKeysCommand` seemed like a perfect solution here. Since public keys of users are available through `https://github.com/<USER>.keys`, we can just download them while trying to connect and override the `authorized_keys` file.

Here is the script we use to get the users from the right teams in our github organisation :
{% highlight bash %}
#!/bin/bash

# Function for downloading keys in parallel
function dl_keys {
  curl -sf https://github.com/${1}.keys
}

# GITHUB_TEAM is the team UID
# GITHUB_TOKEN is the token used for authentication
if [[ ! -z $GITHUB_TEAM && ! -z $GITHUB_TOKEN ]]; then
  users=$(curl -sf https://api.github.com/teams/${GITHUB_TEAM}/members -H "Authorization: token ${GITHUB_TOKEN}"|jq -r '.[].login')

  for i in $users; do
    dl_keys $i &
  done

  wait
else
  echo "Set up GITHUB_TOKEN and GITHUB_TEAM environment variables."
fi
{% endhighlight %}

You will need to create a github authorisation token, to which you can just assign read rights on your organisations and ssh keys.

Once this script is added to the server, add the following lines to your `/etc/ssh/sshd_config` file :
```
AuthorizedKeysCommand <PATH TO SCRIPT>
# Keep in mind the user you use needs to have the environment variables set up -
# nobody won't work here.
AuthorizedKeysCommandUser <USER>
```

# Security
On the security side, the keys being public, there are only a few ways we can be attacked :
- Github being spoofed and serving public keys that are matching the attacker's private key - unlikely
- Our github account being hacked and a malicious user being added to the teams in order to get access

# Fallback
In case of failure of the process, sshd falls back by default to the `authorized_keys` file, where we still have one key stored.

# Update January 2017

To reduce the impact of slow GitHub API responses and to ensure timely responses of the script, a few optimisations can be applied as follows:

- Usage of `--connection-timeout` and `--max-time` options in `curl`
- Usage of a cache on disk with a TTL. A very simple implementation can be
  achieved with `stat` and `date` commands.

The updated script looks as follows:
{% highlight bash %}
#!/bin/bash
cache_file=/path/to/key_cache
function dl_keys {
  curl -m 10 -sf https://github.com/${1}.keys
}

# Function to cache keys to disk
function cache_keys {
  if [[ ! -z $GITHUB_TEAM && ! -z $GITHUB_TOKEN ]]; then
    users=$(curl -m 10 -sf https://api.github.com/teams/${GITHUB_TEAM}/members -H "Authorization: token ${GITHUB_TOKEN}"|jq -r '.[].login')
    for i in $users; do
      dl_keys $i >> $cache_file &
    done
    wait
  else
    echo "Set up GITHUB_TOKEN and GITHUB_TEAM environment variables."
  fi
}

if [[ -f $cache_file ]]; then
  # only re-cache every 5 minutes
  if [ $(( $(date +"%s") - $(stat -c %Y $cache_file) )) -gt 300 ]; then
    rm $cache_file
    cache_keys
  fi
else
  cache_keys
fi
# return contents of cache
if [[ -f $cache_file ]]; then cat $cache_file; fi
{% endhighlight %}

**Note**: `curl` does provide a `-z` option, which sends an `If-Modified-Since` Header. However, we verified and confirmed that the GitHub API never returned a  `HTTP/1.1 304 NOT MODIFIED` response and as such we could not use this option.

[better-ssh]: https://gist.github.com/sivel/c68f601137ef9063efd7
