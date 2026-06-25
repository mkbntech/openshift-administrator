Recommended approach: Shut down the VM cleanly
Pause / Stop

Connect to the OpenShift node:

ssh core@<sno-node>

Perform a graceful shutdown:

sudo shutdown -h now

Verify the VM is powered off:

virsh list --all

You should see the VM in the shut off state.

Resume / Start

Start the VM again:

virsh start <vm-name>

Check status:

virsh list

Wait a few minutes for OpenShift services to come up, then verify:

oc get nodes
oc get co

You should see the node Ready and all cluster operators available.

Alternative: Suspend the VM

KVM/libvirt supports suspending a VM's execution.

Suspend:

virsh suspend <vm-name>

Resume:

virsh resume <vm-name>