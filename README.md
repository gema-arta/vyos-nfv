# vyos-nfv

Packer, Vagrant and others tools to transform VyOS rolling release into a ready to use VyOS image.

## Packer images and Vagrant boxes generation

We use a standard Ubuntu LTE (18.04) for this. First some dependencies :
```
wget https://releases.hashicorp.com/packer/1.4.0/packer_1.4.0_linux_amd64.zip # we fetch a recent packer from their website 
unzip packer_1.4.0_linux_amd64.zip && sudo mv packer /usr/local/bin # make the binary available to the system users
adduser $USER kvm # required for packer to use KVM as normal user, as we use the KVM accelerator
```

### variables

We have put most of the interesting variables into:
```
cat vyos-var.json
```

### rolling release aspects

We provide a python script to generate the latest VyOS rolling release building command : 
```
sudo apt-get install python-bs4 # dependency for the script
python vyos-latest.py
```

The output will be :
```
To build VyOS QCOW image:
packer build -var-file=vyos-var.json -var 'iso_url=https://downloads.vyos.io/rolling/current/amd64/vyos-1.2.0-rolling%2B201905140337-amd64.iso' -var 'iso_checksum=ab014e46588028a021c9adcfb48a32d94dce7f49' -parallel=false vyos.img.json
```

### outputs

We simultaneouly generate the QEMU/libvirt image: 
```
file build_qemu/vyos.img 
build_qemu/vyos.img: QEMU QCOW Image (v3), 17179869184 bytes
```
and the virtualbox image:
```
file build_virtualbox/vyos.ova 
build_virtualbox/vyos.ova: POSIX tar archive
```

Then the packer script is post processing those raw images into Vagrant Boxes:
```
```


### Vagrant based box

As well, after calling packer build, a vagrant-ready image will be made available in the main directory :
```
ls *.box
# vyos-libvirt.box
# vyos-virtualbox.box
```

Those boxes can be added to be ready to use with Vagrant:
```
vagrant box add vyos-libvirt.box --name vyos --force
vagrant box add vyos-virtualbox.box --name vyos --force
```

## Building a router in a couple of lines of Vagrantfile

Using virtualbox provider (default one):
```
vagrant init vyos # this will generate an editable Vagrantfile
vagrant up 
vagrant ssh # to login in the final machine
```

Using libvirt provider:
```
wget https://releases.hashicorp.com/vagrant/2.2.3/vagrant_2.2.3_x86_64.deb
dpkg -i vagrant_2.2.3_x86_64.deb
sudo apt-get install libxslt-dev libxml2-dev libvirt-dev zlib1g-dev ruby-dev 
CONFIGURE_ARGS="with-libvirt-include=/usr/include/libvirt with-libvirt-lib=/usr/lib64" vagrant plugin install vagrant-libvirt
```

Then using the box file generated by packer:
```
vagrant init vyos # this will generate an editable Vagrantfile
vagrant up --provider=libvirt
vagrant ssh # to login in the final machine
```


## Caveats and notes
Note that Vagrant is automatically replacing the insecure ssh key, so it would be wise to configure the new ssh key generated by vagrant (typically available in .vagrant/machines/default/libvirt/private_key) into VyOS using the following :
```
ssh-keygen -y -f .vagrant/machines/default/libvirt/private_key
# this will display the public key part. We just need the key itself : ssh-rsa AAAA.... 
vagrant ssh 
configure
set system login user vagrant authentication public-keys identifier key 'AAAA...'
commit
save
exit
```

## ssh provisioning

One can use the ssh provision of Vagrant to auto configure the VyOS image on vagrant up:
```
/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper begin
/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper set service lldp
/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper commit  
/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper save  
/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper end
```
