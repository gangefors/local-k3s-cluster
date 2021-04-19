# Rancher on a local Kubernetes cluster


## Prerequisite

Ubuntu with `multipass` and `jq` installed.

	sudo apt install jq
	snap install multipass


## Create the cluster nodes

Create at least 2 VMs with multipass using the following commands.

	multipass launch --cpus 2 --mem 2G --disk 10G --cloud-init server-cloud-config.yaml --name server-node &
	multipass launch --cpus 2 --mem 2G --disk 10G --cloud-init agent-cloud-config.yaml --name agent-node1 &
	...
	multipass launch --cpus 2 --mem 2G --disk 10G --cloud-init agent-cloud-config.yaml --name agent-node9 &

> You may change the options to better fit your host system and use case.
> Remove the `&` if you don't want to run them in parallel.

The cloud-init yaml file will provision all VMs with Docker and k3s on
the `server node`.


## Installing k3s on the agent nodes

Since agent nodes need information from the server node we need to
manually install k3s on them.

1. First we need to get the server node IP and a join token. This is the
   information we need to use when installing k3s.

   The commands below will export this information into a local file.

		echo "export server_ip=$(multipass info server-node --format json | jq -r '.info[].ipv4[0]')" > env
		echo "export server_token=$(multipass exec server-node -- sudo cat /var/lib/rancher/k3s/server/node-token)" >> env

2. Copy this file to the agent nodes.

		multipass transfer env agent-node1:env
		...
		multipass transfer env agent-node9:env
		rm env

3. Now we need to open a shell to the node itself.

		multipass shell agent-node1

4. Finally we can run the commands needed to actually install k3s on the
   node and have it join the cluster.

   The first command puts the exported information into the shell's
   environment, then we use it in the installation command.

		eval $(cat env)
		curl -sfL https://get.k3s.io | K3S_URL=https://${server_ip}:6443 K3S_TOKEN=${server_token} sh -s - --docker
		rm env
		exit

Now repeat steps `3` and `4` for all agent nodes.

When you have installed k3s on all nodes we should check the status of
the cluster.

	multipass exec server-node -- sudo kubectl get nodes

	NAME          STATUS   ROLES                  AGE     VERSION
	server-node   Ready    control-plane,master   2m16s   v1.20.5+k3s1
	agent-node1   Ready    <none>                 97s     v1.20.5+k3s1
	agent-node2   Ready    <none>                 57s     v1.20.5+k3s1
	...

If all nodes have status `Ready` all is well.


## Troubleshooting

If anything goes wrong you can always just delete the VM and start from
scratch. Do this with the following commands.

	multipass delete <name>
	multipass purge

Then just start over from the creation of the node.


## References

- https://multipass.run/
- https://cloudinit.readthedocs.io/en/latest/
- https://rancher.com/docs/k3s/latest/en/
