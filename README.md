# KNI IPI Virt

These helper scripts provide a virtualized infrastructure for use with OpenShift baremetal IPI deployment, and then use [OpenShift Baremetal Deploy Ansible Installer](https://github.com/openshift-kni/baremetal-deploy/tree/master/ansible-ipi-install) to deploy a cluster on that virtualized infrastructure.  They do the following:

1. Prepare the provisioning host for OCP deployment (required packages, firewall, etc)
2. Start DHCP and DNS containers for the OCP baremetal network
3. Set up NAT forwarding and masquerading to allow the baremetal network to reach an external routable network
4. Create VMs to serve as the cluster's masters and workers
5. Create virtual BMC endpoints for the VMs
6. Clone the [OpenShift Baremetal Deploy Ansible Installer](https://github.com/openshift-kni/baremetal-deploy/tree/master/ansible-ipi-install) and prepare it for use with the virtualized infrastructure
7. Execute the aforementioned Ansible playbook

## Prerequisites

1. Provisioning host machine must have 3 NICs on separate VLANs: 
   - External/SSH network
   - Provisioning network
   - Baremetal network
2. Provisioning host machine must be RHEL 8 or CentOS 8
3. If RHEL 8, an active subscription is required
4. A non-root user must be available to execute the scripts and the Ansible playbook.  You could add one like so:
    ```sudo useradd kni
    echo "kni ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/kni
    sudo chmod 0440 /etc/sudoers.d/kni
    sudo su - kni -c "ssh-keygen -t rsa -f /home/kni/.ssh/id_rsa -N ''"```
5. `sudo dnf install -y make git`
6. Copy your OpenShift pull secret to your non-root user's home directory (i.e. `/home/kni`) and call it `pull-secret.txt` (this location is ultimately configurable, however -- see below)

## Bundled Usage

1. Clone the repo to your provisioning host machine and go to the directory:
    ```git clone https://github.com/redhat-nfvpe/kni-ipi-virt.git
    cd kni-ipi-virt```
2. Set your environment variables in `common.sh`.  These values and their purpose are described in the file.
3. `make all`
4. To remove the VMs, DNS and DHCP containers, use `make clean`

## Isolated Usage

1. Clone the repo to your provisioning host machine and go to the directory:
    ```git clone https://github.com/redhat-nfvpe/kni-ipi-virt.git
    cd kni-ipi-virt```
2. Set your environment variables in `common.sh`.  These values and their purpose are described in the file.
3. Execute `prep_host.sh`, which requires the following variables to be set in `common.sh`:

- `BM_BRIDGE`
- `BM_GW_IP`
- `BM_INTF`
- `DNS_IP`
- `PROV_BRIDGE`
- `PROV_INTF`

4. Assuming steps above have been completed, the individual DNS, DCHP and VM bash scripts can be utilized alone to make use of their atomic functionality.

### DNS

Create a CoreDNS container to provide DNS on your baremetal network.  The following variables are required to be set in `common.sh`:

- `API_VIP`
- `BM_GW_IP`
- `CLUSTER_DOMAIN`
- `CLUSTER_NAME`
- `DNS_IP`
- `DNS_VIP`
- `EXT_DNS_IP`
- `INGRESS_VIP`
- `PROJECT_DIR`

`./dns/start.sh`
`./dns/stop.sh`

### DHCP

Create a dnsmasq container to provide DHCP on your baremetal network.  The following variables are required to be set in `common.sh`:

- `BM_GW_IP`
- `BM_INTF`
- `CLUSTER_DOMAIN`
- `CLUSTER_NAME`
- `DNS_IP`
- `MASTER_BM_MAC_PREFIX`
- `PROJECT_DIR`
- `WORKER_BM_MAC_PREFIX`

`./dhcp/start.sh`
`./dhcp/stop.sh`

### VMs

Create a certain number of VMs for use with an OCP deployment.  The following variables are required to be set in `common.sh`:

- `CLUSTER_NAME`
- `LIBVIRT_STORAGE_POOL`
- `MASTER_BM_MAC_PREFIX`
- `MASTER_CPUS`
- `MASTER_MEM`
- `MASTER_PROV_MAC_PREFIX`
- `MASTER_VBMC_PORT_PREFIX`
- `NUM_MASTERS`
- `NUM_WORKERS`
- `PROJECT_DIR`
- `WORKER_BM_MAC_PREFIX`
- `WORKER_CPUS`
- `WORKER_MEM`
- `WORKER_PROV_MAC_PREFIX`
- `WORKER_VBMC_PORT_PREFIX`

`./vms/prov-vms.sh`
`./vms/clean-vms.sh`

## Troubleshooting

1. If you are unable to start the DNS container because of an error message like so...
   
   `Error: error from slirp4netns while setting up port redirection: map[desc:bad request: add_hostfwd: slirp_add_hostfwd failed]`

   ...try stopping/removing all containers and killing all remaining `slirp4nets` processes, and then try to start the container again.  Sometimes `podman` fails to clean up the `slirp4netns` forwarding processes when it stops/removes the DNS container. 