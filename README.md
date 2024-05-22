# Ansible role `cloud_init` 

The **cloud_init** Ansible role that acts as a provisioner for creating multiple virtual machines using cloud instances quickly. This role is also available as part of the **techiesanjay.libvirt_infra** content collection on Ansible Galaxy.

This role leverages on libvirt and qemu packages along with cloud-init to create VMs. Cloud-init is the **industry standard multi-distribution method for cross-platform cloud instance initialization**. It is supported across all major public cloud providers, provisioning systems for private cloud infrastructure, and bare-metal installations. This role utilizes cloud-init for instantiating virtual machines using libvirt packages. The role is optimized in many ways:

 - Allows the root user to login with a password over a network
 - Optionally allows for creating a cloud user with sudo privileges and locked for password access. Can be used with ssh_authorized_keys
 - Prevents multiple downloads of the same cloud image
 - Prevents re-creation of VMs by stalling the role if they already exist. This behavior can be changed
 - For slow connections the cloud image can be downloaded and put in a local folder to prevent downloading from the web

## Requirements

The role is dependent on the following content collections:

 - community.libvirt
 - ansible.posix

This role also requires the following:

 - qemu-kvm, qemu-img, genisoimage and libguestfs-tools package
 - active libvirt storage pool and network previously provisioned as required by the VMs
 
The above requirements can be easily met if the **libvirt_setup** role is used.

## Role Variables
This role requires a set of variables along with the dictionary  **cloud_vms_info**  in role/defaults that require to be overridden to create a set of VMs. The dictionary uses a key that acts as the VMs name and a set of VM related parameters to be used on a per VM basis. 

### cloud_vms_info.<VM_NAME> dictionary variables
This dictionary is used initialize parameters for a specific VMs instance that need to be created using the **cloud_init** role.

| Variable | Comments |
|--|--|
| vm_image_link | URL for a specific cloud image |
| vm_local_hostname | Hostname assigned to the VM during the cloud-init process. |
| vm_fqdn | Fully qualified domain name of the Virtual machine. Used by cloud-init |
| vm_root_passwd | Password to set for the root user during the cloud-init process |
| vm_custom_cloud_user | Optional custom cloud user to be created during cloud-init process. Password access will be locked. User is sudo enabled |
| vm_ssh_authorized_keys | list of ssh public keys that will be applied to the root and the optional cloud  user during the cloud-init process |
| vm_os_type | Type of the OS being installed. i.e. **windows** or **linux**
| vm_os_variant | Variant of the OS being installed. Run the **virt-install --os-variant list** command to see a set of supported OS variants.
| vm_memory | Memory required by the VM.|
| vm_cpu | Number of vCPUs required by the VM|
| vm_root_disk_size | Total size of the block device|
| vm_static_ip | Static IP to be assigned to the VM. (optional)|
| vm_netmask | Subnet mask to be used with the static IP. Must be set if static IP is being used|
| vm_gateway | The default gateway that the VM will use to access external networks. Must be set if static IP is used|
| vm_dns_search_domain | DNS search domain to be used in ethernet script of the VM |
| vm_dns1 | IP address of the first DNS server. **vm_dns2** and **vm_dns3** can be used to hold IP address of secondary and tertiary DNS servers.|
| vm_nm_controlled_network: 'yes'| Can have values '**yes**' or '**no**'. 'Yes' is used compulsorily for RHEL8 family as they only use the NetworkManager service. For RHEL7 family you can set this to 'no'|
| vm_additional_nics | Defines additional network interface required by the VM. This is defined as a list where each element is the name of the libvirt network to use. There should not be more than 3 elements in this list.|
| vm_additional_disks | A list that defines additional block devices. Each list element indicates the capacity if the block device |

### Other role variables
|Variable|Comments  |
|--|--|
| vm_recreate | Defaults to a value of **False** for safety reasons so that VMs are not recreated if they are already existing. This causes your role to stall with a message. Can be set to **True** to delete the VM and recreate it again|
| cloud_images_dir | Defaults to the name **cloud_images**. Will be created in the same place as the playbook that calls the role, if it doesn't exist. If this folder is already existing then the role tries to rsync the folder content to the **/var/lib/libvirt/images** path |
| cloud_storage_path | Path where VM boot disk and other block device based resources are stored. |
| cloud_network | Libvirt network to be used by cloud images. This defaults to the default libvirt network called **default**. |
| cloud_image_download_timeout| Max time to download cloud image. Defaults to 300 seconds. you will need to increase this value for slower internet connections. |
| cloud_image_download_poll | Polls for task completion every 10 seconds by default. |


#### Example. 
Creates a VM called test1 that uses a static IP address on the first nic and creates an additional nic that makes use of the default libvirt network.
```Yaml
cloud_vms_info:
  test1:
    vm_image_link: http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-2009.qcow2
    vm_local_hostname: test1
    vm_fqdn: test1.example.com
    vm_root_passwd: letmein
    vm_os_type: linux
    vm_os_variant: rhel7.0
    vm_memory: 1024
    vm_cpu: 1
    vm_root_disk_size: 10G
    vm_static_ip: 192.168.122.250
    vm_netmask: 255.255.255.0
    vm_gateway: 192.168.122.1
    vm_dns1: 192.168.122.2
    vm_nm_controlled_network: yes
    vm_additional_nics:
      - default
```
> **NOTE**: If the above list of dictionaries is not used to override the cloud_vms_info dictionary in the defaults/main.yml while executing the role then the role will create two VMs called **test1** and **test2** based on the existing dictionary in role defaults. One will function using a static IP address the other will take a DHCP lease from the default libvirt network. Both VMs will use disks located in the default storage pool.

#### Example. 
 Creates a VM called test2 that uses a DHCP lease from the libvirt network. This VM is created with two additional block device of 10G each.

```Yaml
cloud_vms_info:
  test2:
    vm_image_link: http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-2009.qcow2
    vm_local_hostname: test2
    vm_fqdn: test2.example.com
    vm_root_passwd: letmein
    vm_os_type: linux
    vm_os_variant: rhel7.0
    vm_memory: 1024
    vm_cpu: 1
    vm_root_disk_size: 10G
    vm_additional_disks:
      - 10G
      - 20G
    vm_nm_controlled_network: 'no'
```

## How to use this role?

#### Playbook to create additional libvirt storage and network
```Yaml
- name: Starting with cloud-init configuration
  hosts: localhost
  become: yes
  vars_files:
    - 'configs/password.yml'
    - 'configs/cloudinit-config.yml'
    
  vars:
    ansible_become_password: "{{ sudo_password }}"

  roles:
    - cloud_init

  tasks:
    # Waiting for SSH availability on all VMs in this infrastructure.
    - name: Wait for SSH to be available on VM/s
      wait_for:
        host: "{{ item.value.vm_static_ip | ipaddr('address') }}"
        state: started
        delay: 0
        timeout: 300
        port: 22
      become: false
      loop: "{{ lookup('dict',cloud_vms_info, wantlist=True) }}"
      loop_control:
        label: "{{item.key}}"
```
For additional examples lookup the **tests** directory within the role.

#### configs/cloudinit-config.yml
```Yaml
cloud_vms_info:
  utility:
    vm_image_link: https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2
    vm_local_hostname: admin
    vm_fqdn: admin.cephstorage.io
    vm_root_passwd: letmein
    vm_os_type: linux
    vm_os_variant: rhel8.0
    vm_memory: 2048
    vm_cpu: 2
    vm_root_disk_size: 10G
    vm_static_ip: 192.168.5.2
    vm_netmask: 255.255.255.0
    vm_gateway: 192.168.5.1
    vm_dns1: 192.168.5.1
    vm_nm_controlled_network: 'yes'
    vm_additional_nics:
      - ceph-cluster-nw   
  ceph-node1:
    vm_image_link: https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2
    vm_local_hostname: node1
    vm_fqdn: node1.cephstorage.io
    vm_root_passwd: letmein
    vm_os_type: linux
    vm_os_variant: rhel8.0
    vm_memory: 4096
    vm_cpu: 4
    vm_root_disk_size: 10G
    vm_additional_disks:
      - 10G
      - 10G
      - 10G
      - 10G
      - 10G
    vm_static_ip: 192.168.5.3
    vm_netmask: 255.255.255.0
    vm_gateway: 192.168.5.1
    vm_dns1: 192.168.5.1
    vm_nm_controlled_network: 'yes'
    vm_additional_nics:
      - ceph-cluster-nw 
  ceph-node2:  
    vm_image_link: https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2  
    vm_local_hostname: node2  
    vm_fqdn: node2.cephstorage.io  
    vm_root_passwd: letmein  
    vm_os_type: linux  
    vm_os_variant: rhel8.0  
    vm_memory: 4096  
    vm_cpu: 2  
    vm_root_disk_size: 10G  
    vm_additional_disks:  
      - 10G  
      - 10G  
      - 10G  
    vm_static_ip: 192.168.5.4  
    vm_netmask: 255.255.255.0  
    vm_gateway: 192.168.5.1  
    vm_dns1: 192.168.5.1  
    vm_nm_controlled_network: 'yes'  
    vm_additional_nics:  
      - ceph-cluster-nw
```

#### configs/password.yml
```Yaml
# CHANGE this to match the host on which this role will be executed
sudo_password: GobbleDeGook
```

## Dependencies
No role dependencies have been established. However, as mention earlier this role is dependent on specific packages that you will need to manually install. Therefore, you are encouraged to use the **libvirt_setup** role to handle package management and pre-flight check to work effectively with the **cloud_init** role.

## License
BSD

## Author
**Sanjay Singh**
sanjay.kr.singh@gmail.com
> Written with [StackEdit](https://stackedit.io/).


