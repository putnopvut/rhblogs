Basic outline: What? 

Looking around the internet, it's pretty easy to find high-quality tutorials
on basics of OVN. However, when it comes to more advanced topics, I sometimes
feel like the amount of ready information is lacking. In this tutorial, we'll
examine dynamic addressing in OVN. You will learn about IP address management
(IPAM) options in OVN and how to apply them.

One of the first things you'll learn when starting with OVN is how to create
logical switches and logical switch ports. You'll probably see something like
this:

````
ovn-nbctl ls-add sw
ovn-nbctl lsp-add sw sw-p1
ovn-nbctl lsp-set-addresses sw-p1 00:ac:00:ff:01:01 192.168.0.1
````

It's pretty simple. But it requires you to manually keep track of IP addresses
for the switch ports.

If you dig a bit deeper, you can find tutorials on the web that describe how to
set up OVN to provide IP addresses using DHCP. This saves you some configuration
steps on the VMs, but it doesn't help any on the OVN side. This is because you
still have to specify an IP address on the logical switch port.

But is there some way that you can actually have OVN dynamically assign
addresses to switch ports? If you scrounge through the ovn-nb manpage, you might
be able to piece together the way to do it, but here it is all in one tutorial!
FUCK YEAH!

Let's start with the relevant options you can set, and then we'll look at some
examples that use these options. All of these are set as "options" on logical
switches.

- subnet: This is an IPv4 subnet, specified as a network address and mask. For
  example, 10.0.0.0/8 or 10.0.0.0/255.0.0.0

- exclude\_ips: This is a list of IPv4 addresses that should not be assigned to
  switch ports. You can either comma-separate individual addresses, or you can
  specify a range of addresses.

- ipv6\_prefix: This is an IPv6 network address of 64 bits. If you provide a
  longer address size than 64 bits, then those past the first 64 bits are
  ignored. The IPv6 address provided on each switch port is an EUI64 address
  using the specified prefix and the MAC address of the port.

Once you have these options set on your switch, it's then a matter of setting
your switch ports up to make use of these options. You can do this in one of two
ways. Way #1:

````
ovn-nbctl lsp-set-addresses sw-p2 00:ac:00:ff:01:01 dynamic
````

Way #2:

````
ovn-nbctl lsp-set-addresses sw-p2 dynamic
```` 

With way #1, you specify the MAC address, and with way #2, you allow for OVN to
allocate the MAC address for you.

So let's go forth with an example where we do some basic setup:

````
ovn-nbctl ls-add dyn-switch
ovn-nbctl set Logical_Switch other_config:subnet=192.168.0.0/24 \
          other_config:ipv6_prefix=fdd6:8cf5:9dc2:c30a
ovn-nbctl lsp-add dyn-switch port1 
ovn-nbctl lsp-set-addresses port1 dynamic
ovn-nbctl lsp-add dyn-switch port2
ovn-nbctl lsp-set-addresses port2 "00:ac:00:ff:01:01 dynamic"
````

With this setup, we've created a logical switch called "dyn-switch" and created
two ports called (inventively) "port1" and "port2".

So what happens when you do this? Let's take a look at the database contents for
the logical switch ports at the moment:

````
$ovn-nbctl list logical_switch_port

\_uuid               : 2b2170dd-6129-47e1-bedf-89438dd1636f
addresses           : ["00:ac:00:ff:01:01 dynamic"]
dhcpv4_options      : []
dhcpv6_options      : []
dynamic_addresses   : "00:ac:00:ff:01:01 192.168.0.3"
enabled             : []
external\_ids        : {}
name                : "port2"
options             : {}
parent\_name         : []
port_security       : []
tag                 : []
tag_request         : []
type                : ""
up                  : false

\_uuid               : 1672414b-62db-42e9-afa2-f5b8ab8a7c7e
addresses           : [dynamic]
dhcpv4_options      : []
dhcpv6_options      : []
dynamic_addresses   : "0a:00:00:00:00:01 192.168.0.2"
enabled             : []
external_ids        : {}
name                : "port1"
options             : {}
parent\_name         : []
port\_security       : []
tag                 : []
tag\_request         : []
type                : ""
up                  : false

````

<UH>What? Why aren't there IPv6 addresses?</UH>

Notice the "dynamic_addresses" for the two switch ports. This database column is
automatically populated by ovn-northd based on IPAM configuration on the logical
switch. For port 1, we specified "dynamic" for the addresses, so OVN created a
MAC address and an IP address for us. You can recognize OVN-allocated MAC
addresses because they always start with "0a". Notice also that the IP address
is 192.168.0.2. Port 2, on the other hand, has the MAC address that we
specified. And then it gets the next IP address in the sequence, 192.168.0.3.

One question you might ask is, "why didn't port 1 get assigned 192.168.0.1?"
The reason is that the first address in the subnet is reserved for the logical
router that the switch connects to. So if we were to connect this switch to a
logical router, and the switch port connected to the logical router used a
dynamic address, then that switch port would get assigned 192.168.0.1. 

Let's take a closer look into the exclude_ips option. The pattern we have seen
is that the IP addresses assigned to each switch port is monotonically
allocated, starting with the second address in the subnet. Let's set up an
exclude_ips and then set up a third port and see that it is actually applied as
we would expect.

````
ovn-nbctl set Logical_Switch other_config:exclude_ips=192.168.0.3..192.168.0.100
````

And now let's set up a third port on the logical switch
````
ovn-nbctl lsp-add dyn-switch port3
ovn-nbctl lsp-set-addresses port3 dynamic
````

Based on the pattern we had previously seen, we might expect for port3 to have
IP address 192.168.0.3. However, we set up exclude_ips to include 192.168.0.3
through 192.168.0.100. So let's see what port3 actually has.

````
add the shizz here
````

This can be a useful feature to use if you want to mix static and dynamically
assigned IP addresses on your logical switch. Or if you have some other weird
dumb setup that requires excluding certain IP addresses.

With all this, you should have the tools you need to set up IP addresses on your
logical switches without the need to keep track of assigned addresses in your
application.

But there's more to this than what I have presented here. In a follow-up
blogpost, we'll look at some of the downsides in the IPAM implementation of OVN,
and we will delve into fixes that are coming in the upcoming version of OVN.

Love,
Mark
