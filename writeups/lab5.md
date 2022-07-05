Lab 5 Writeup
=============

My name: Zhijing Hu

<del>My SUNet ID: [your sunetid here]</del>

<del>I collaborated with: [list sunetids here]</del>

<del>I would like to thank/reward these classmates for their help: [list sunetids here]</del>

This lab took me about 4 hours to do. <del>I [did/did not] attend the lab session.</del>

### Program Structure and Design of the `NetworkInterface`:

#### Additional Private Members

```c++
    //! arp_table maps the IPv4 raw address to an Ethernet address with a time stamp
    std::unordered_map<uint32_t, std::pair<size_t, EthernetAddress>> _arp_table;

    //! record the current time
    size_t _time;

    //! queued frames because of temporarily unkown dst ethernet address
    //  records its time stamp to avoid resend many ARP requests in a short time
    std::unordered_map<uint32_t, std::pair<size_t, std::queue<EthernetFrame>>> _queued_frame;

    //! helper function to construct and send an arp frame
    //  if we want to send a ARP request, then `dst_eth` need not to be filled
    //  otherwise `dst_eth` MUST be filled
    void send_arp(uint16_t opcode, uint32_t dst_ip, std::optional<EthernetAddress> dst_eth);
```

#### Methods

##### `void NetworkInterface::send_datagram(const InternetDatagram &dgram, const Address &next_hop)`

1. Produce an `EthernetFrame`, fill in the necessary information to its head and payload except the `dst` Ethernet address;
2. Find the corresponding destination Ethernet address from the `_arp_table` and check if it goes stale. If it exists and it's fresh, fill it into the frame and send the frame directly;
3. Otherwise we have to resolve the `dst` Ethernet address first by broadcasting an ARP request message first:
   1. Queue the frame to the `_queued_frame[next_hop_ip].second`;
   2. Check the time stamp in `_queued_frame[next_hop_ip].first`, if `_time` is bigger than it by 5000 ms, then send/resend an ARP request message and update the time stamp

##### `optional<InternetDatagram> NetworkInterface::recv_frame(const EthernetFrame &frame)`

1. If the frame is an IPv4 datagram and aims at the interface, parse it and return it;
2. Else it is an ARP message. Learn the mapping from the `sender` fields first. Check if there are any queued frames that can be sent now and send them if they exist, then:
   1. If it's a reply message, learn the mapping from `target` fields. Check if there are any queued frames that can be sent now and send them if they exist;
   2. If it's a request message and the interface is its target, then send a reply message to the sender.

##### `void NetworkInterface::tick(const size_t ms_since_last_tick)`

Only update the `_time`.

### Implementation Challenges:

None.

### Remaining Bugs:

NONE.

- Optional: I had unexpected difficulty with: None

- Optional: I think you could make this lab better by: None

- Optional: I was surprised by: 

  Even though I erased a stale mapping in `std::unordered_map` in the `tick()` method, it continues existing. I do not know why because I add some debugging output and ensure the deletion of that stale mapping. 

  Finally I changed my implementation to avoid this. And I believe the new implementation is more efficient because it does not need to iterate through the whole mappings to find out the stale mappings every time the `tick()` is called.

- Optional: I'm not sure about: None
