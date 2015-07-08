# Vagrant

- [http://www.vagrantup.com/](http://www.vagrantup.com/) Vagrant homepage
- [http://vagrantmanager.com/](http://vagrantmanager.com/) OSX Utility to manage vagrant boxes
- [https://atlas.hashicorp.com/boxes/search](https://atlas.hashicorp.com/boxes/search) Vagrant box repository

## Set up a new Vagrant VM

### Find a box

Find the box you want to use for your project on [atlas.hashicorp.com](https://atlas.hashicorp.com/boxes/search). I'm using **ubuntu/trusty64**.

### Init

From the root of your project, run `vagrant init`:

```bash
vagrant init ubuntu/trusty64
```

This creates your **Vagrantfile**.

### Vagrantfile

Your generated **Vagrantfile** will have very little code. It's mostly just comments that indicate how you can configure your box.

#### Specify box image

You specifiy what box you'd like installed. This is the guest virtual machine that's installed into your VM provider (VirtualBox, Parallels, Docker, etc).

```ruby
config.vm.box = "ubuntu/trusty64"
```
#### Forward ports

You can forward ports from your guest box to your host. In the following, you'd be able to access port 80 on the guest from port 8000 on your host:

```ruby
config.vm.network "forwarded_port", guest: 80, host: 8000
```

You can use this to get at things like Apache, MySQL, etc.

#### Box IP address

Give the box an IP address that the can be used to access the guest from the host. With this, you don't need to forward ports from the guest to the host.

```ruby
config.vm.network "private_network", ip: "192.168.33.10"
```

#### Mount a shared resource

Vagrant will automatically mount the folder it is in on the box at **/vagrant**. You can mount additional folders:

```ruby
config.vm.synced_folder "/var/www/html", "/usr/share"
```

##### Disable default share

You can disable the default root share by adding the share to your Vagrantfile and setting `disabled` to `true`:

```ruby
config.vm.synced_folder ".", "/vagrant",
    disabled: true
```

##### Ownership and permisssions

You cannot change the user, group, or permissions for the share from the guest.

You can set the ownership in your Vagrantfile:

```ruby
config.vm.synced_folder ".", "/vagrant",
    owner: "vagrant", group: "www-data"
```

Then to change the permissions, set them from the host with `chmod`.


#### Provider options

You have the abillity to specify the allotted memory and available CPUs.

You can do it statically:

```ruby
config.vm.provider "virtualbox" do |v|
    v.cpus = 4
    v.memory = 1024
end
```

Or dynamically based on the host (found this online):

```ruby
# Give VM 1/4 system memory & access to all cpu cores on the host
config.vm.provider "virtualbox" do |v|
    host = RbConfig::CONFIG['host_os']
    if host =~ /darwin/
        v.cpus = `sysctl -n hw.ncpu`.to_i
        # sysctl returns Bytes and we need to convert to MB
        v.memory = `sysctl -n hw.memsize`.to_i / 1024 / 1024 / 4
    elsif host =~ /linux/
        v.cpus = `nproc`.to_i
        # meminfo shows KB and we need to convert to MB
        v.memory = `grep 'MemTotal' /proc/meminfo | sed -e 's/MemTotal://' -e 's/ kB//'`.to_i / 1024 / 4
    end
end
```

### Up

Now that your **Vagrantfile** is set up, start it up:

```bash
vagrant up
```

This will download the box, set it up in your provider (VirtualBox, Parallels, etc), set up SSH, etc.

It'll store some info in a new directory, **.vagrant**.

**NOTE:** The **.vagrant** directory should not be under source control.

## Other basic vagrant commands

We've already covered `init` and `up`.

- `vagrant halt` stops the vm.
- `vagrant reload` stops and reloads the vm.
- `vagrant suspend` suspends the vm.
- `vagrant resume` will wake up a vm that's been suspended. `up` will do the same thing.
- `vagrant destroy` will delete the vm from your provider (VirtualBox, Parrallels, etc).

## Provisoning

Provisionig is important so that you can easily recreate the environment your appliction needs.

### Shell

Provisioning with a shell script simply runs the provided script from the guest OS.

```ruby
config.vm.provision "shell", path: "provision/script.sh"
```

A sample shell script used to provision CentOS as a LAMP stack:

[https://gist.github.com/flackend/8f0bd097edf87f5462eb](https://gist.github.com/flackend/8f0bd097edf87f5462eb)

### Ansible

Ansible is another provisioning method that has more of a learning curve than shell provisioning, but is more powerful.

```ruby
config.vm.provision "ansible" do |ansible|
  ansible.sudo = true
  ansible.playbook = "ansible/playbook.yml"
end
```

Sample ansible playbook to provision a LAMP stack:

[https://github.com/flackend/sample-lamp-playbook](https://github.com/flackend/sample-lamp-playbook)

## Share

If you create an Atlas account ([atlas.hashicorp.com](https://atlas.hashicorp.com/)), you can share your box/connect to it remotely.

Login to your atlas account:

```bash
vagrant login
```

Share the box:

```bash
vagrant share --ssh
```

You'll get a generated name and URL. You can connect to the URL in your browser and connect to the name with `vagrant connect`.

The `--ssh` is optional, but allows you to login over SSH.

## Connecting to the host from the guest

- [How to connect with host PostgreSQL from vagrant virtualbox machine (StackOverflow)](http://stackoverflow.com/questions/19933550/how-to-connect-with-host-postgresql-from-vagrant-virtualbox-machine)

You may want to connect to your host from your guest machine. You can do that by retreiving the default gateway:

```bash
netstat -rn | grep "^0.0.0.0 " | cut -d " " -f10
```

From the StackOverflow answer:

> By running `netstat -rn` you can get the default gateway and then use that ip address in your config file of your application.

This is helpful if you want to connect to the database on your host, for instance. That'll give you better performance.
