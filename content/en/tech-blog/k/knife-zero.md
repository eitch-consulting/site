---
title: Local usage of Chef with knife-zero
date: 2024-06-01T07:19:50-03:00
draft: false
language: en
featured_image: ../assets/images/featured/featured-progress-chef-logo.png 
summary:
  Sometimes you don't need or want a dedicated server for configuration management with Chef, and this is
  completely possible, you can use a knife plugin called `knife-zero` to simulate a local server with
  chef local mode and work everything within a repository.
categories: Tech-Blog
tags:
- automation
- chef
- configuration management
type: tech-blog
---

Sometimes you don't need or want a dedicated server for configuration management with Chef, and this is
completely possible, you can use a knife plugin called `knife-zero` to simulate a local server with
Chef local mode and work everything within a repository.

This can also be called a `push based` system instead of the default `pull based` chef server. I'm not
going to address the differences, pros and cons because that's not the scope of this article. I'm assuming
you already know these and is interested in doing this with Chef.

How this works:

1. Install the `chef-workstation` tools, including the `knife` CLI utility;

2. Install the `knife` plugin named `knife-zero`;

3. You create a local repository (i.e. using git) containing the basic chef structure;

4. Instead of using `knife` to do things like: upload cookbooks, data bags, that will interact with a 
   chef server, you use `knife-zero` and it will automatically run a local, temporary chef-server with
   all content from the repository.

In this article, I'm also assuming that you have basic knowledge about configuration management and Chef, and  
I'll focus on the technical aspect of setting up `knife-zero` and using it.

## Installing tools

Here we'll use the official tools provided by Chef. It's also good to mention that after the Chef trademark
usage was changed to a more commercial approach, there's a community behind [CINC](https://cinc.sh/) which stands
for *CINC is not Chef* that provides the same tools I'll use here. So if you're inside an commercial companny
that is not looking for a commercial/paid product, use it. It's the same thing!

1. [Download and install the chef-workstation package](https://docs.chef.io/workstation/install_workstation/)

    This package includes all the necessary tools to work locally with chef: an embedded Ruby interpreter, Ruby
    gems for `chef`, `chef-client`, `chef-zero`, `knife`, `ohai`, `cookstyle`, `foodcritic`, `kitchen`, and various
    other tools.

    In my case, I'm using Fedora Linux 40, so the equivalent chef-workstation version would be Red Hat Enterprise
    Linux 8.x or 9.x. You can check the package's latest version on the
    [https://docs.chef.io/release_notes_workstation/](release notes) page.

    Following the instructions, I download and installed the package:

    ```bash
    wget https://packages.chef.io/files/stable/chef-workstation/24.4.1064/el/9/chef-workstation-24.4.1064-1.el9.x86_64.rpm
    sudo dnf localinstall chef-workstation-24.4.1064-1.el9.x86_64.rpm
    ```

2. Install the `knife-zero` Ruby gem using the following command:

    ```bash
    chef gem install knife-zero
    ```

    This will run the `gem install` command but inside the embedded Ruby interpreter from the chef-workstation, and for
    the user you're running (we're not using sudo here). You can check all gems installed on this embedded ruby with the
    command:

    ```bash
    chef gem list 
    ```

    Notice that `knife-zero` should be on the list!

3. Test if `knife-zero` is working by running the command:

    ```bash
    knife zero
    ```

    The command should return:

    ```plain
     ** ZERO COMMANDS **
     knife zero apply QUERY (options)
     knife zero bootstrap [SSH_USER@]FQDN (options)
     knife zero chef_client QUERY (options) | It's same as converge
     knife zero converge QUERY (options)
     knife zero diagnose # show configuration from file
     ```
     
     That's it! All you need should be installed and ready to be used.

## Create and configure a repository

In this article I'll create a repository from scratch. But you can also use any already-created Chef repository and
configure it to use knife-zero.

Create the base repository by running:

```bash
chef generate repo chef-repo
cd chef-repo
```

This creates a skeleton reposity at directory `chef-repo`. We'll also create a dummy cookbook to test it later:

```bash
cd cookbooks
chef generate cookbook testing
```

And the following content on `chef-repo/cookbooks/testing/recipes/default.rb`:

```ruby
file '/tmp/testing' do
  owner 'root'
  group 'root'
  mode '0640'
  content "this was created by chef for #{node['hostname]}"
end
```

Back to the repository's root directory, let's create a specially crafted knife configuration to always use
a Chef local server and `knife-zero`. Create a `chef-repo/config.rb` file with this content:

```ruby
# configuration file for local mode chef

local_mode true
chef_zero.enabled true
config_log_level :error

node_name 'admin'
client_key File.expand_path(__dir__) + '/.chef/admin.pem'

cookbook_path [
  File.expand_path(__dir__) + '/cookbooks',
]

knife[:chef_repo_path] = File.expand_path(__dir__)
knife[:ssh_user] = 'rocky'
knife[:ssh_identity_file] = '~/.ssh/id_ed25519'
knife[:ssh_timeout] = 10
knife[:use_sudo] = true
knife[:listen] = true
knife[:editor] = '/usr/bin/vim'
knife[:log_level] = 'error'
knife[:ssh_attribute] = 'knife_zero.host'
knife[:secret_file] = File.expand_path(__dir__) + '/.chef/encrypted_data_bag_secret'
knife[:encrypted_data_bag_secret] = File.expand_path(__dir__) + '/.chef/encrypted_data_bag_secret'

knife[:allowed_automatic_attributes] = %w(
  fqdn
  os
  os_version
  hostname
  ipaddress
  roles
  recipes
  ipaddress
  platform
  platform_version
  cloud
  cloud_v2
  chef_packages
  ohai_time
)

silence_deprecation_warnings %w(chef-18)
```

Some notes about this configuration:

- Replace `knife[:ssh_user] = 'rocky'` with the SSH username you use to connect to servers. In this case I'm using
  the `rocky` username for the Rocky Linux distribution. This can also be replaced while running the knife CLI later.

- Replace `knife[:ssh_identity_file] = '~/.ssh/id_ed25519'` with the SSH key you use to connect to servers. This key
  should be valid for the username (in this example, `rocky`) by allowing it in `authorized_keys` on the server.

Create the admin client. Inside the `chef-repo` directory, run:

```bash
mkdir .chef
knife client create admin -f .chef/admin.pem
# a text editor will be shown, change the "admin" value to "true", save and exit
chmod 0640 .chef/admin.pem
```

Create the encrypted data bag secret. Inside the `chef-repo` directory, run:

```
openssl rand -base64 512 | tr -d '\r\n' > .chef/encrypted_data_bag_secret
chmod 0640 .chef/encrypted_data_bag_secret
```

...And we're ready to roll with everything configured.

## Bootstrap new nodes

With the tools, repository and configuration ready, now it's time to bootstrap a new node with Chef and use it. To do
this, you should have access to a server using the configured SSH username and key (in this article, `rocky` and
`~/.ssh/id_ed25519`. This remote user also **needs to gain root using sudo** in the new node.

With the IP address of the server you want to bootstrap, run:

```bash
knife zero bootstrap <ip> --node-name my-new-server.example.com -r 'recipe[testing]'
```

This command will:

1. Run a Chef local server with the contents of the local repository;

2. Make a bridge between this server and the new node;

3. Install the Chef and its tools (i.e. `chef-client`) inside the node;

4. Run a `chef-client` with the specified `recipe[testing]` runlist. This is the testing cookbook we wrote earlier.

Sample output (yeah, in this sample I used Ubuntu :P):

```plain
Connecting to 192.168.124.101 using ssh
WARNING: Performing legacy client registration with the validation key at ...
WARNING: Remove the key file or remove the 'validation_key' configuration option from your config.rb (knife.rb) to use more secure user credentials for client registration.
Bootstrapping 192.168.124.101
 [192.168.124.101] -----> Installing Chef Omnibus (stable/18)
downloading https://omnitruck.chef.io/chef/install.sh
  to file /tmp/install.sh.4232/install.sh
trying wget...
 [192.168.124.101] ubuntu 22.04 x86_64
Getting information for chef stable 18 for ubuntu...
downloading https://omnitruck.chef.io/stable/chef/metadata?v=18&p=ubuntu&pv=22.04&m=x86_64
  to file /tmp/install.sh.4236/metadata.txt
 [192.168.124.101] trying wget...
 [192.168.124.101] sha1 f6037852e758977da9d23d87206348522a7506a0
sha256  57fd4c44ab37d9e311cbdd9dceeca73dc1ebf342b6cfb4e75632f0b7fcd4336d
url     https://packages.chef.io/files/stable/chef/18.4.12/ubuntu/22.04/chef_18.4.12-1_amd64.deb
version 18.4.12
 [192.168.124.101] 
 [192.168.124.101] downloaded metadata file looks valid...
 [192.168.124.101] downloading https://packages.chef.io/files/stable/chef/18.4.12/ubuntu/22.04/chef_18.4.12-1_amd64.deb
  to file /tmp/install.sh.4236/chef_18.4.12-1_amd64.deb
 [192.168.124.101] trying wget...
 [192.168.124.101] Comparing checksum with sha256sum...
 [192.168.124.101] Installing chef 18
installing with dpkg...
 [192.168.124.101] Selecting previously unselected package chef.
 [192.168.124.101] (Reading database ... 64405 files and directories currently installed.)
 [192.168.124.101] Preparing to unpack .../chef_18.4.12-1_amd64.deb ...
 [192.168.124.101] Unpacking chef (18.4.12-1) ...
 [192.168.124.101] Setting up chef (18.4.12-1) ...
 [192.168.124.101] Thank you for installing Chef Infra Client! For help getting started visit https://learn.chef.io
 [192.168.124.101] Starting the first Chef Infra Client Client run...
 [192.168.124.101] +---------------------------------------------+
✔ 2 product licenses accepted.
+---------------------------------------------+
 [192.168.124.101] Chef Infra Client, version 18.4.12
 [192.168.124.101] Patents: https://www.chef.io/patents
 [192.168.124.101] Infra Phase starting
 [192.168.124.101] Creating a new client identity for my-new-server.example.com using the validator key.
 [192.168.124.101] Resolving cookbooks for run list: ["testing"]
 [192.168.124.101] Synchronizing cookbooks:
 [192.168.124.101] - testing (0.1.0)
 [192.168.124.101] Installing cookbook gem dependencies:
Compiling cookbooks...
 [192.168.124.101] Loading Chef InSpec profile files:
 [192.168.124.101] Loading Chef InSpec input files:
 [192.168.124.101] Loading Chef InSpec waiver files:
 [192.168.124.101] Converging 1 resources
 [192.168.124.101] Recipe: testing::default
 [192.168.124.101] * file[/tmp/testing] action create
 [192.168.124.101] - create new file /tmp/testing
 [192.168.124.101] - update content in file /tmp/testing from none to 7f7e25
 [192.168.124.101] --- /tmp/testing     2024-06-01 13:03:50.901599562 +0000
 [192.168.124.101] +++ /tmp/.chef-testing20240601-4323-slc8kl   2024-06-01 13:03:50.901599562 +0000
 [192.168.124.101] @@ -1 +1,2 @@
 [192.168.124.101] +this was created by chef for pgsql
 [192.168.124.101] - change mode from '' to '0640'
 [192.168.124.101] - change owner from '' to 'root'
 [192.168.124.101] - change group from '' to 'root'
 [192.168.124.101] Running handlers:
 [192.168.124.101] Running handlers complete
 [192.168.124.101] Infra Phase complete, 1/1 resources updated in 02 seconds
```

If we look in the server, our `testing` recipe created the file `/tmp/testing`:

```bash
cat /tmp/testing
```

Output:

```plain
this was created by chef for my-new-server
```

After the bootstrap is done, you don't need to bootstrap again. The node was also created in the local
repository with these two files:

- `clients/my-new-server.example.com.json` - the public key for this node
- `nodes/my-new-server.example.com.json` - the inventory for this node (generated by chef and `ohai`)

You can also use any `knife` command in the repository root directory to manage this node, example:

```bash
knife node list
```

Output:

```plain
my-new-server.example.com
```

```bash
knife node show my-new-server.example.com
```

Output:

```plain
Node Name:   my-new-server.example.com
Environment: _default
FQDN:        my-new-server
IP:          192.168.124.101
Run List:    recipe[testing]
Roles:       
Recipes:     testing, testing::default
Platform:    ubuntu 22.04
Tags:
```

## Managing nodes

Since we don't need to bootrap the node, we can run the same recipe using `knife zero converge`, like this:

```bash
knife zero converge 'name:my-new-server.example.com'
```

This will do almost the same thing as bootstraping, but it will not need to install the chef-client since it's
already installed.

The `name:my-new-server.example.com` is a search string. In this example, I'm telling to run chef-client only on
the server named `my-new-server.example.com`, but I can use any other search string:

```bash
# run on all nodes
knife zero converge 'name:*'

# run only on ubuntu servers:
knife zero converge 'platform:ubuntu'

# run only on production environment, centos servers:
knife zero converge 'chef_environment:production AND platform:centos'

# run only on servers running the webserver cookbook:
knife zero converge 'recipes:testing'

# and so on...
```

To check the search syntax, consult [this documentation for knife search](https://docs.chef.io/workstation/knife_search/).

# Conclusion and references

And that's it! You can use chef and knife as you use in any other environment. You won't need to always do
a `knife cookbook upload` because every time you run `knife zero`, it will read the current repository and
serve from that. That's no *upstream server* to upload anything to.

Also, there's no *catch* in writing cookbooks. Everything will just run as if you have a server. You don't need to
programatically check for parameters like when you use `chef-solo` or `dry-run`. It will work out-of-the-box.

References:

- [Chef Infra Client (executable) - Run in local mode](https://docs.chef.io/ctl_chef_client/#run-in-local-mode)
- [knife search](https://docs.chef.io/workstation/knife_search/)
- [config.rb](https://docs.chef.io/workstation/config_rb/)
- [config.rb optional settings](https://docs.chef.io/workstation/config_rb_optional_settings/)
