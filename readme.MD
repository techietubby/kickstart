Begin by creating a new host in /etc/hosts or adding a DNS entry.

192.168.0.47    podman.persephone.local   podman
192.168.0.47    ipa.example.com

Edit ./inventory add add the new host.

Create/copy a vars and templates file.

./vars/podman.yml
./templates/podman.j2

Edit the ./vars file to point to the new host and configure the IP Address and partition sizes etc.

# ansible-playbook provision-vm.yml -e vm=podman