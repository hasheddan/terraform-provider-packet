---
layout: "packet"
page_title: "Packet: packet_device"
sidebar_current: "docs-packet-resource-device"
description: |-
  Provides a Packet device resource. This can be used to create, modify, and delete devices.
---

# packet_device

Provides a Packet device resource. This can be used to create,
modify, and delete devices.

~> **Note:** All arguments including the `root_password` and `user_data` will be stored in
 the raw state as plain-text.
[Read more about sensitive data in state](/docs/state/sensitive-data.html).


## Example Usage

Create a device and add it to cool_project

```hcl
resource "packet_device" "web1" {
  hostname         = "tf.coreos2"
  plan             = "t1.small.x86"
  facilities       = ["ewr1"]
  operating_system = "coreos_stable"
  billing_cycle    = "hourly"
  project_id       = local.project_id
}
```

Same as above, but boot via iPXE initially, using the Ignition Provider for provisioning

```hcl
resource "packet_device" "pxe1" {
  hostname         = "tf.coreos2-pxe"
  plan             = "t1.small.x86"
  facilities       = ["ewr1"]
  operating_system = "custom_ipxe"
  billing_cycle    = "hourly"
  project_id       = local.project_id
  ipxe_script_url  = "https://rawgit.com/cloudnativelabs/pxe/master/packet/coreos-stable-packet.ipxe"
  always_pxe       = "false"
  user_data        = data.ignition_config.example.rendered
}
```

Create a device without a public IP address, with only a /30 private IPv4 subnet (4 IP addresses)

```hcl
resource "packet_device" "web1" {
  hostname         = "tf.coreos2"
  plan             = "t1.small.x86"
  facilities       = ["ewr1"]
  operating_system = "coreos_stable"
  billing_cycle    = "hourly"
  project_id       = local.project_id
  ip_address {
      type = "private_ipv4"
      cidr = 30
  }
}
```

Deploy device on next-available reserved hardware and do custom partitioning.

```hcl
resource "packet_device" "web1" {
  hostname         = "tftest"
  plan             = "t1.small.x86"
  facilities       = ["sjc1"]
  operating_system = "ubuntu_16_04"
  billing_cycle    = "hourly"
  project_id       = local.project_id
  hardware_reservation_id = "next-available"
  storage = <<EOS
{
  "disks": [
    {
      "device": "/dev/sda",
      "wipeTable": true,
      "partitions": [
        {
          "label": "BIOS",
          "number": 1,
          "size": "4096"
        },
        {
          "label": "SWAP",
          "number": 2,
          "size": "3993600"
        },
        {
          "label": "ROOT",
          "number": 3,
          "size": "0"
        }
      ]
    }
  ],
  "filesystems": [
    {
      "mount": {
        "device": "/dev/sda3",
        "format": "ext4",
        "point": "/",
        "create": {
          "options": [
            "-L",
            "ROOT"
          ]
        }
      }
    },
    {
      "mount": {
        "device": "/dev/sda2",
        "format": "swap",
        "point": "none",
        "create": {
          "options": [
            "-L",
            "SWAP"
          ]
        }
      }
    }
  ]
}
EOS
}
```

## Argument Reference

The following arguments are supported:

* `hostname` - (Required) The device name
* `project_id` - (Required) The ID of the project in which to create the device
* `operating_system` - (Required) The operating system slug. To find the slug, or visit [Operating Systems API docs](https://www.packet.com/developers/api/operatingsystems), set your API auth token in the top of the page and see JSON from the API response.
* `facilities` - List of facility codes with deployment preferences. Packet API will go through the list and will deploy your device to first facility with free capacity. List items must be facility codes or `any` (a wildcard). To find the facility code, visit [Facilities API docs](https://www.packet.com/developers/api/facilities), set your API auth token in the top of the page and see JSON from the API response.
* `plan` - (Required) The device plan slug. To find the plan slug, visit [Device plans API docs](https://www.packet.com/developers/api/plans), set your auth token in the top of the page and see JSON from the API response.
* `billing_cycle` - (Required) monthly or hourly
* `user_data` (Optional) - A string of the desired User Data for the device.
* `ipxe_script_url` (Optional) - URL pointing to a hosted iPXE script. More
  information is in the
  [Custom iPXE](https://www.packet.com/developers/docs/servers/operating-systems/custom-ipxe/)
  doc.
* `always_pxe` (Optional) - If true, a device with OS `custom_ipxe` will
  continue to boot via iPXE on reboots.
* `hardware_reservation_id` (Optional) - The `full ID` of the hardware reservation where you want this device deployed, or `next-available` if you want to pick your next available reservation automatically.
  Please be careful when using hw reservation UUID and `next-available` together for the same pool of resevations. It might happen that the reservation which Packet API will pick as `next-available` is the reservation which you refer with UUID in another packet_device resource. If that happens, and the packet_device with the UUID is created later, resource creation will fail because the reservation is already in use (by the resource created with `next-available`). To workaround this, have the `next-available` resource  [explicitly depend_on](https://learn.hashicorp.com/terraform/getting-started/dependencies.html#implicit-and-explicit-dependencies) the resource with hw reservation UUID, so that the latter is created first. For more details, see [issue #176](https://github.com/terraform-providers/terraform-provider-packet/issues/176).
* `storage` (Optional) - JSON for custom partitioning. Only usable on reserved hardware. More information in in the [Custom Partitioning and RAID](https://www.packet.com/developers/docs/servers/key-features/cpr/) doc.
  * Please note that the disks.partitions.size attribute must be a string, not an integer. It can be a number string, or size notation string, e.g. "4G" or "8M" (for gigabytes and megabytes).
* `tags` - Tags attached to the device
* `description` - Description string for the device
* `project_ssh_key_ids` - Array of IDs of the project SSH keys which should be added to the device. If you omit this, SSH keys of all the members of the parent project will be added to the device. If you specify this array, only the listed project SSH keys will be added. Project SSH keys can be created with the [packet_project_ssh_key][https://www.terraform.io/docs/providers/packet/r/project_ssh_key.html] resource.
* `network_type` (Optional) - Network type of device, used for [Layer 2 networking](https://www.packet.com/developers/docs/network/advanced/layer-2/). Allowed values are `layer3`, `hybrid`, `layer2-individual` and `layer2-bonded`. If you keep it empty, Terraform will not handle the network type of the device.
* `ip_address` (Optional) - A list of IP address types for the device (structure is documented below). 
* `wait_for_reservation_deprovision` (Optional) - Only used for devices in reserved hardware. If set, the deletion of this device will block until the hardware reservation is marked provisionable (about 4 minutes in August 2019).
* `force_detach_volumes` (Optional) - Delete device even if it has volumes attached. Only applies for destroy action.

The `ip_address` block has 3 fields:

* `type` - One of [`private_ipv4`, `public_ipv4`, `public_ipv6`]
* `cidr` - CIDR suffix for IP address block to be assigned, i.e. amount of addresses.
* `reservation_ids` - String of UUID of [IP block reservations](https://www.terraform.io/docs/providers/packet/r/reserved_ip_block.html) from which the public IPv4 address should be taken.

You can supply one `ip_address` block per IP address type. If you use the `ip_address` you must always pass a block for `private_ipv4`.

To learn more about using the reserved IP addresses for new devices, see the examples in the [packet_reserved_ip_block](https://www.terraform.io/docs/providers/packet/r/reserved_ip_block.html) documentation.

## Attributes Reference

The following attributes are exported:

* `access_private_ipv4` - The ipv4 private IP assigned to the device
* `access_public_ipv4` - The ipv4 maintenance IP assigned to the device
* `access_public_ipv6` - The ipv6 maintenance IP assigned to the device
* `billing_cycle` - The billing cycle of the device (monthly or hourly)
* `created` - The timestamp for when the device was created
* `deployed_facility` - The facility where the device is deployed.
* `description` - Description string for the device
* `hardware_reservation_id` - The ID of hardware reservation which this device occupies
* `hostname`- The hostname of the device
* `id` - The ID of the device
* `locked` - Whether the device is locked
* `network` - The device's private and public IP (v4 and v6) network details. When a device is run without any special network configuration, it will have 3 networks: 
  * Public IPv4 at `packet_device.name.network.0`
  * IPv6 at `packet_device.name.network.1`
  * Private IPv4 at `packet_device.name.network.2`
  Elastic addresses then stack by type - an assigned public IPv4 will go after the management public IPv4 (to index 1), and will then shift the indices of the IPv6 and private IPv4. Assigned private IPv4 will go after the management private IPv4 (to the end of the network list).
  The fields of the network attributes are:
  * `address` - IPv4 or IPv6 address string
  * `cidr` - bit length of the network mask of the address
  * `gateway` - address of router
  * `public` - whether the address is routable from the Internet
  * `family` - IP version - "4" or "6"
* `operating_system` - The operating system running on the device
* `plan` - The hardware config of the device
* `ports` - Ports assigned to the device
  * `name` - Name of the port (e.g. `eth0`, or `bond0`)
  * `id` - ID of the port
  * `type` - Type of the port (e.g. `NetworkPort` or `NetworkBondPort`)
  * `mac` - MAC address assigned to the port
  * `bonded` - Whether this port is part of a bond in bonded network setup
* `project_id`- The ID of the project the device belongs to
* `root_password` - Root password to the server (disabled after 24 hours)
* `ssh_key_ids` - List of IDs of SSH keys deployed in the device, can be both user and project SSH keys
* `state` - The status of the device
* `tags` - Tags attached to the device
* `updated` - The timestamp for the last time the device was updated
