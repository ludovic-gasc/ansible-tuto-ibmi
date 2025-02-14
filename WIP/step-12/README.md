# Ansible tutorial: Migrating to roles

Now that our playbook is done, let's refactor everything! We'll replace
our plays with roles. Roles are just a new way of organizing files but
bring interesting features. I won't go into great lengths here, since
they're listed in
[Ansible's documentation](http://www.ansibleworks.com/docs/playbooks_roles.html#id5),
but my favorite is probably roles dependencies: role B can depend on
another role A. Thus, when applying role B, role A will automatically be
applied too. We'll see this in the [next
chapter](https://github.com/ludovic-gasc/ansible-tuto-ibmi/tree/master/step-13),
but for now, let's refactor our playbook to use roles.

## Roles structures

Roles add a bit of "magic" to Ansible: they assume a specific file
organization. While there is a suggested layout regarding roles, you can
organize things the way you want using includes. However, role's conventions
help building modular playbooks, and housekeeping will be much simpler.
Rubyists would call this "convention over configuration".

The file layout for roles looks like this:

```none
roles
  |
  |_some_role
       |
       |_defaults
       |   |
       |   |_main.yml
       |   |_...
       |
       |_files
       |   |
       |   |_file1
       |   |_...
       |
       |_handlers
       |   |
       |   |_main.yml
       |   |_some_other_file.yml
       |   |_ ...
       |
       |_meta
       |   |
       |   |_main.yml
       |   |_some_other_file.yml
       |   |_ ...
       |
       |_tasks
       |   |
       |   |_main.yml
       |   |_some_other_file.yml
       |   |_ ...
       |
       |_templates
       |   |
       |   |_template1.j2
       |   |_...
       |
       |_vars
           |
           |_main.yml
           |_some_other_file.yml
           |_ ...
```

The files named `main.yml` are not mandatory. However, when they exist,
roles will add them to the play automatically.
You can use this file to include other tasks, handlers, ... in the play.
We'll see that in a minute.

Note that there is also a `vars` and a `meta` directory. `vars` is used
when you want to put a bunch of variables regarding the roles. However,
I don't like setting vars in roles (or plays) directly. I think variables
belong to configuration, while plays are the structure. In other words,
I see plays and roles as a factory, and data as inputs to this factory.
So I really prefer to have "data" (e.g. variables) outside roles and
play. This way, I can share my roles more easily, without worrying about
exposing too much about my servers. But that's just a personal
preference. Ansible just lets you do it the way you want.

But you have some vars that you hardly want to change. For instance, if
you have a role for nginx that pulls the .deb package from a PPA, you
might want to add the PPA address in `vars/main.yml`. It is something
that you _can_ configure, but that will be mostly static 99% of the
time. Using `vars` will let you pull out this information out of your
role, making it more generic. But really, this is a matter of taste.

However, for real vars (e.g. things you would like to use in a
configuration file generated by a template), you can set defaults for
roles, and this is a recommended practice. Using sane defaults ensures
your role always work.
For instance, you could set the number of pre-forked servers for your
apache server. The best place to put the defaults is... you guessed it,
the `defaults` directory.

The `meta` directory is where you can add dependencies, and it's really
a neat feature. We'll see that later.

Note that roles sit in the `roles` directory, which is also cool since
it will reduce top level ansible playbook clutter. But you can configure
Ansible to use an alternate directory to store role (see [`roles_path`
variable in `ansible.cfg`](http://docs.ansible.com/intro_configuration.html#roles-path)).
This way you can setup a 'central place ' for all your roles, and use
them in all your playbooks.

## Creating the Apache role

Ok, now that we know the required layout, we can create our apache role
from our apache playbook.

The steps required are really simple:

- create the roles directory and apache role layout
- extract the apache handler into `roles/apache/handlers/main.yml`
- move the apache configuration file `awesome-app` into
  `roles/apache/files/`
- create a role playbook

### Creating the role layout

This is what has been done to convert step-11 apache files into a role:

```bash
mkdir -p step-12/roles/apache/{tasks,handlers,files}
```

Now we need to copy the tasks from `apache.yml` to `main.yml`, so this
file looks like this:

```yaml
- name: Installs necessary packages
  apt:
    pkg: ["apache2", "libapache2-mod-php", "git"]
    state: latest
    update_cache: true
  tags:
    - apache

...

- name: Deactivates the default ssl virtualhost
  command: a2dissite default-ssl
  notify:
    - restart apache
```

The file is not fully reproduced, but it is exactly the content of
`apache.yml` between `tasks:` and `handlers:`.

Note that we also have to remove references to `files/` and `templates/`
directories in tasks. Since we're using the roles structure, Ansible
will look for them in the right directories.

### Extracting the handler

We can extract the handlers part and create
`step-12/roles/apache/handlers/main.yml`:

```yaml
- name: restart apache
  service:
    name: apache2
    state: restarted
```

### Moving the configuration file

As simple as:

```bash
cp step-11/files/awesome-app step-12/roles/apache/files/
```

At this point, the apache role is fully working, but we need a way to
invoke it.

### Create a role playbook

Let's create a top level playbook that we'll use to map hosts and host
groups to roles. We'll call it `site.yml`, since our goal is to have our
site-wide configuration in it. While we're at it, we'll include
`haproxy` in it too:

```yaml
- hosts: web
  roles:
    - apache

- hosts: haproxy
  roles:
    - haproxy
```

That wasn't too hard.

Now let's create the haproxy role:

```bash
mkdir -p step-12/roles/haproxy/{tasks,handlers,templates}
cp step-11/templates/haproxy.cfg.j2 step-12/roles/haproxy/templates/
```

Instead of creating the haproxy role directory structure using mkdir, it's
possible to use ansible-galaxy:

```bash
ansible-galaxy --offline init step-12/roles/haproxy
cp step-11/templates/haproxy.cfg.j2 step-12/roles/haproxy/templates/
```

then extract the handler, and remove reference to `templates/`.

We can try out our new playbook with:

```bash
ansible-playbook -i step-12/hosts step-12/site.yml
```

If eveything goes well, we should end up with a happy "PLAY RECAP" like
this one:

```none
host0          : ok=4    changed=0    unreachable=0 failed=0
host1          : ok=9    changed=5    unreachable=0 failed=0
host2          : ok=9    changed=5    unreachable=0 failed=0
```

You may have noticed that running all roles in site.yml can take a long
time.  What if you only wanted to push changes to web?  This is also
easy, with the limit flag:

```bash
ansible-playbook -i step-12/hosts -l web step-12/site.yml
```

This concludes our migration to roles. It was quite easy, and adds a
bunch of features to our playbook that we'll use in a future step.

In
[step-13](https://github.com/ludovic-gasc/ansible-tuto-ibmi/tree/master/step-13),
we will see how we can use tags to select which parts of our playbook we
want to run.
