---
layout: post
title: Cooking a Web-Server with Chef-Solo [DRAFT]
description: "Cooking a Web-Server with Chef-Solo"
modified: 2013-10-25
tags: [chef, chef-solo, web server, devops]
comments: false
share: true
---



Stasis is bad. There is nothing agile about statis.
When a component falls into statis, I'll let it be, and it'll carry on doing what it does.
Until it breaks, that is. Everything falls apart until it can be fixed.

When I first SSH'd into a remote box on the other side of the world,
I felt like a hacker. I would pain-stakingly wait the 250ms RTT and configure the box. I had to.
If I did not, everything would not work. Over time, the server fell into statis, and then it broke.
Development stopped. Growth stopped. Changed stopped.

Chef offers a means to declare your server architecture, and cook it.
Learning to cook with Chef feels like learning a DSL.
You have to learn a language which operates at a layer of abstraction above what you actually want to do.
I despise learning DSLs because of the shortness of their lifespans,
however regarding Chef,
learning it doesn't feel as esoteric as learning Puppet,
and I find the process a necessity now.

If you are like me, and like to experiment, then often things can go fubar.
It is comforting and encouraging to know that everything can be reconstructed in 15 minutes.




# Concepts
Cooking is simple. Chef offers a means of declaring a server architecture, while being agnostic to the underlying platform.
Simple. however, the concepts and the roles they play were the biggest stumbling block I experienced.


Chef was built to distribute to multiple nodes
Chef-Solo does not need a dedicated Chef Server
Chef-Solo is a handy means to putting up a web server. There are a few quirks between full blown Chef and Chef-Solo.


Knife is a tool. This part is clear from the metaphor.
Intuitively, I thought that knife was a tool used by Chef to perform the cook.
Wrong. Knife is a tool that you use to instruct Chef to do it's job.
Knife is a nice name, but a bit counter intuitive.

Knife Solo is what you will use to perform solo cooks.

The server needs to be prepared before you can cook



# Ruby
You will need Ruby. Chef is written in Ruby. RVM is a means to manage different versions of Ruby.
RVM has been criticed for achieving too much, and that RBEnv is more focussed.

RVM works for me for now. Have some faith give curl root priviledges:


{% highlight bash %}
\curl -L https://get.rvm.io | bash -s stable --ruby
    curl -L get.rvm.io | bash -s stable
{% endhighlight %}

The root priviledges are so that it can make changes to openssl. Should I not be a bit more paranoid here?


And then you have to source it.

{% highlight bash %}
source /Users/indika/.rvm/scripts/rvm
{% endhighlight %}


Bundler is awesome (according to the community.) Use it to manage the gems required for the client side of the Chef.
Use Gem to obtain Bundler.

{% highlight bash %}
gem install bundler
{% endhighlight %}

Now that the Ruby dependencies have been set up, we are ready to create a kitchen.



# Creating a Kitchen

The kitchen is the structure you create on the client side before you cook your server.

## ~/.chef

This is the default location where Chef / Knife (?) looks for machine specific settings.
This folder is not on the server, as I initially imagined. This is my *knife.rb* configuration file.

{% highlight ruby %}
knife[:provisioning_path] = "/home/root/solo"

cookbook_path [
    './cookbooks',
    'site-cookbooks']

role_path     "roles"
data_bag_path "data_bags"
encrypted_data_bag_secret "#{ENV['HOME']}/.chef/motion_secret"
{% endhighlight %}


This file might be necessary for knife solo because Chef was built to run on it's own dedicated server.


## A workspace for the Kitchen

Create a directory for your kitchen.

### Bundler and the Gemfile

Use a Gemfile to specify the dependencies for Chef.


{% highlight ruby %}
source 'https://rubygems.org'

gem 'knife-solo'
gem 'knife-solo_data_bag'
gem 'librarian'
gem 'multi_json'
gem 'foodcritic'
{% endhighlight %}



And then use Bundler to obtain these dependencies.


{% highlight bash %}
bundle install
{% endhighlight %}


I'm not sure exactly why Bundler is better than Gem to manage dependencies.
But I do know Bundler can maintain the specific version number required for the application (Chef).
And it also obtains the gems required by the required gems, and so forth.
And I do know that it simply just work.


### Scaffolding


Choose a directory and use Knife to create a scaffolding.

{% highlight bash %}
knife solo init .
{% endhighlight %}

Your kitchen should look like:


{% highlight text %}
├── Gemfile
├── Gemfile.lock
├── cookbooks
│   └── - public cookbooks get loaded in here
│       -
├── data_bags
│   └── - secret stuff
├── nodes
│   └── - configuration of particular nodes
├── roles
│   └── - common configurations that can be applied
│         across multiple nodes
└── site-cookbooks
    └── - cookbooks that you have written
{% endhighlight %}


### Librarian-Chef and the Cheffile

Cookbooks contain recipes that instruct Chef how to do stuff.
One of the reasons Chef appeals to me the most is because of the wealth of open source cookbooks available.

I've noticed three different managers for cookbooks:

* Librarian-Chef
* Berkshelf
* knife-github-cookbooks

I started with Librarian-Chef and currently see no good reason to change.

Create the scaffolding for a Cheffile


{% highlight bash %}
librarian-chef init
{% endhighlight %}





This is my Cheffile

{% highlight ruby %}
site 'http://community.opscode.com/api/v1'

cookbook 'runit'
cookbook 'cron', :git => 'git://github.com/opscode-cookbooks/cron.git'
cookbook 'database', :git => 'git://github.com/opscode-cookbooks/database.git'
cookbook 'logwatch', :git => 'git://github.com/opscode-cookbooks/logwatch.git'
cookbook 'nginx', :git => 'git://github.com/opscode-cookbooks/nginx.git'
cookbook 'phantomjs', :git => 'git://github.com/customink-webops/phantomjs.git'
cookbook 'python', :git => 'git://github.com/opscode-cookbooks/python.git'
cookbook 'postgresql', :git => 'git://github.com/opscode-cookbooks/postgresql.git'
cookbook 'user', :git => 'git://github.com/fnichol/chef-user.git'
{% endhighlight %}


And fetch your cookbooks:

{% highlight bash %}
librarian-chef install --clean
{% endhighlight %}






### Solo.rb
This seems to be deprecated.
solo.rb found, but since knife-solo v0.3.0 it is not used any more



# Bootstrapping the Server

Before you can cook your server, it has to be ready for cooking.
In the ideal world, Chef can and will completely determine the architecture of the server. However, when it comes to practicalities, the server still needs to be bootstrapped before it can be cook. It broadly involves:

- setting up a SSH connection, so that you  (meaning the software running on your computer) can securely communicate with it, and
- installing stuff like unattended upgrades, and any obviously required packages (I'm not going to be idealistic here on not having dependencies outside of the Chef configuration)
- the existence of Chef specific libraries.
- the existence of the secret file to decrypt data bags (more on this later)

## Configuring SSH

SSH is pretty much the standard when it comes to opening a secure communication channel with a server.
I'll assume you know how to set it up.

There has been a recent debate on whether it is [a bad idea](http://www.adayinthelifeof.nl/2012/03/12/why-putting-ssh-on-another-port-than-22-is-bad-idea) or
[a good idea](http://www.danielmiessler.com/blog/putting-ssh-another-port-good-idea)
to use the a non-default port. I prefer to use a non-default port because I cannot deal with the noise of random hackers.


## Remotely installing Chef


{% highlight bash %}
knife solo prepare root@motion -p 2222

… Setting up chef (11.6.2-1.ubuntu.12.04)
{% endhighlight %}


This installs Ruby, RubyGems and Chef on target mach. Chef needs to be installed on the server


## Create a secret file and SCP it

Vulnerable pieces should be encrypted. More about this listen. For now, simply generate a secret key and upload it via SCP.


{% highlight bash %}
openssl rand -base64 512 > ~/.chef/motion_secret
scp -P 2222 ~/.chef/motion_secret root@motion:/root/.chef/motion_secret
{% endhighlight %}




# Nodes

The .json configuration files is where the definition of a node (node) exists.
This is a good place to start interrogating Chef.

It contains of a bunch of node specific data. The most pertinent is the run list. The run list is where the action happens, and it appears the order of items in the list, is the order in which the action happens.


# Structure of a Cookbook

Create a cookbook


{% highlight bash %}
knife cookbook create entity -r md
{% endhighlight %}



{% highlight text %}
entity
├── CHANGELOG.md
├── README.md
├── attributes
├── definitions
├── files
│   └── default
├── libraries
├── metadata.rb
│   └── - information about the creator
│       - has some depends statements
├── providers
├── recipes
│   └── default.rb
├── resources
└── templates
    └── default

{% endhighlight %}






## Includes and stuff like this

include recipe[rvm::user] in your run_list and add a user hash to the user_installs attribute list.

eg

{% highlight ruby %}

node['dvm']['user_installs'] = [
  { 'user'          => 'wiggle bottom',
    'default_ruby'  => 'rbx',
    'rubies'        => ['1.9.2', '1.8.7']
  }
]

{% endhighlight %}







# Databags

Knife solo uploads cookbooks, roles and data bags onto the target node.
This was not obvious to me at first.
It was possible that it would only sending instructions.

Databags are annoying, yet feel necessary. Errors only become apparent during the cook. However, they seem important and my struggled attempt to justify them are as follows:

- The Chef server could be compromised
- Can manage data bags in source control without plain text
- Perhaps the reason behind data bags is because private information can be intercepted between the Chef server and the node.

Nevertheless, I will use them to wire good security habit.

The basic idea is that keys are stored in .json files and encrypted versions are sent to the Chef server.

An annoying point to bear in mind is, the ID of the data bag must match the name of the file, minus the .json name. Otherwise, the convention fails!


## Encryption Keys

Use openssl to generate an encryption key.

{% highlight bash %}

openssl rand -base64 512

{% endhighlight %}



And use this key to encrypt the data bag.

{% highlight bash %}

knife solo data bag create indika bob
--secret-file /Users/indika/.chef/encrypted_data_bag_secret
--data-bag-path data_bags
-e vim
--json-file data_bags/indika/_bob.json

{% endhighlight %}




- It takes a secret file.
- A path to place the bag
- A preferred editor
- And a source bag

Notice how this creates the structure for me. And I fully specify the source.


This is the strategy:
Have a folder called data\_bags\_plain

Inside there, have an example folder

Now, there are two concepts,
the folder, and the file name.

I suppose the first name refers to the site-cookbook - well it could.

{% highlight json %}
{
    "id": "users",
    "users": [
        {
            "user_name": "publish",
            "tag": "publish",
            "comment": "User for publishing content",
            "use_ssh": true,
            "ssh_key": "secret ssh key"
        }
    ]
}
{% endhighlight %}



{% highlight ruby %}

secret = Chef::EncryptedDataBagItem.load_secret("/root/.chef/motion_secret")

entity_users_bag = Chef::EncryptedDataBagItem.load("entity", "users", secret)
entity_users_bag['users'].each do |user_item|
    puts user_item['user_name']
end

{% endhighlight %}




# Roles

Roles are a means of applying common functionality to multiple nodes. I'm only cooking one node, so moving on...

Can define using JSON or the Ruby DSL.




# Next


Somethings are harder to cook than others.
The Ruby community seems to be very splintered.


Ansible and Salt exist. So does Puppet.
What about OpenStack. People talk about OpenStack on OpenStack.

Chef for cooking a server.
And Docker for distributing programs.

Berkshelf instead of Librarian.