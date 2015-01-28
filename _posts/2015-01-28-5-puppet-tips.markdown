---
layout: post
title:  "5 Puppet Tips"
tags: puppet dev-ops
permalink: /:year/:month/:day/:title
---

# 1. Add a comment to managed configuration files

If you have configuration files managed by Puppet -- e.g. `my.cnf` for MySQL, `php.ini` for PHP,
or Nginx files -- be sure to add a comment at the top of the files informing readers of those files
that they are managed by Puppet and any changes will be overwritten. This will prevent someone
unfamiliar with the server's configuration or with Puppet from wasting hours wondering why their
changes disappear.

Example Nginx file:

{% highlight nginx %}
# This file is managed by Puppet. All changes will be overwritten.

server {
    # ...
}
{% endhighlight %}

# 2. Have an `apply` file

Create a file called `apply` that runs `puppet apply` with all the correct parameters for your
environment. This will save you typing time, prevent typos, and documents how to run the Puppet
apply command. Add `$*` at the end of the command to preserve any parameters you use when calling
`./apply (parameters)`.

Example:

{% highlight bash %}
#!/bin/bash
sudo puppet apply manifests/site.pp \
    --modulepath modules/:~/.puppet/modules $*
{% endhighlight %}

Note: in `--modulepath`, a colon (`:`) works like the Unix $PATH -- it is a seperator for multiple
directories.

Make sure your apply file is executable with `chmod +x apply`.

# 3. Use `puppet-lint`

`puppet-lint` is a tool that's checks your manifests for violations from the standard Puppet style.
To install it, on Debian/Ubuntu run `sudo apt-get install puppet-lint`, or on CentOS run
`sudo yum install puppet-lint`. Like the `apply` file above, you should create a `lint` file that runs
`puppet-lint` with your desired configuration. For my projects, I disable some linting options, like
the 80-character line length limit.

I configure my `lint` files to exit with status code 1 (which indicates failure in Unix terminology)
if there are any errors. This allows you to use it as a Git pre-commit hook.

An example `lint` file: 

{% highlight bash %}
#!/bin/bash

files=$( find . -type f | grep \.pp$ )
exit_status=0

for f in $files; do
    errors=$( puppet-lint $f --no-80chars-check --no-arrow_alignment-check --no-documentation-check )
    if [[ $errors != '' ]]; then
        echo "$f:"
        echo $errors
        echo
        exit_status=1
    fi
done

exit $exit_status
{% endhighlight %}

To use this file as a Git pre-commit hook, run `ln -s ../../lint .git/hooks/pre-commit`. Make sure your
lint file is executable with `chmod +x lint`.

# 4. Define Puppet's execution $PATH

By default, to run commands in `exec` blocks, you must use full paths, for example:

{% highlight puppet %}
exec { '/bin/cp config.sample config': }
{% endhighlight %}

To avoid having to write the full path to each executable like `/bin/cp`, define Puppet's execution $PATH
at the start of your base manifest:

{% highlight puppet %}
Exec {
    path => '/bin:/usr/bin'
}
{% endhighlight %}

# 5. Save lines with arrays

Instead of this verbose manifest code:

{% highlight puppet %}
package { 'mysql-server':
    ensure => present,
}

package { 'mysql-client':
    ensure => present,
}
{% endhighlight %}

You can combine these into one statement:

{% highlight puppet %}
package { ['mysql-server', 'mysql-client']:
    ensure => present,
}
{% endhighlight %}

This also works with files and services:

{% highlight puppet %}
file { ['/home/mysite/project', '/home/mysite/logs', '/home/mysite/uploads']:
    ensure => directory,
}
{% endhighlight %}
