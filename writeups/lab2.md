Lab 2 Writeup
=============

My name: Zhijing Hu

This lab took me about 4 hours to do.

### Program Structure and Design of the `TCPReceiver` and wrap/unwrap routines:

See my code in[`libsponge/wrapping_intergers.cc`](./libsponge/wrapping_integers.cc).

#### For wrap

This is relatively easy. Basically, I just need to return the `(n+isn)%UINT32_MAX`. 

But a corner case should be handled carefully: `UINT32_MAX` is $2^{32}-1$, so when the low-32-bits of `n-isn` equals `UINT32_MAX`( i.e. `isn=0` && `n=UINT32_MAX`), it should be set to `UINT32_MAX` instead of 0.

```c++
WrappingInt32 wrap(uint64_t n, WrappingInt32 isn) {
	if (static_cast<uint32_t>((n + isn.raw_value())) == UINT32_MAX) {
        return WrappingInt32(UINT32_MAX);
    }
    return WrappingInt32{static_cast<uint32_t>((n + isn.raw_value())) % UINT32_MAX};
}
```

##### For unwrap

`uint64_t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64_t checkpoint);`.

Because `n - isn` == `absolute sequence number % UINT32_MAX`, here's the pseudo-code:

```c++
uint64_t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64_t checkpoint) {
    uint64_t asn;
    asn <= n;
    while (asn < isn + checkpoint) {
        add asn by (1ul << 32);
    }
    // first wrap
    // 0--+--UINT32_MAX--+--2*UINT32_MAX  -> absolute sequence number 
    //    |              |
    //   lsn             n
    if (ret < isn.raw_value()) {
        return ret + (1ul << 32) - isn;
    }
    // no wrap
    // 0------+-------------+-------UINT32_MAX  -> absolute sequence number 
    //        |             |
    //       lsn            n
    if (ret <= static_cast<uint64_t>(UINT32_MAX) + isn.raw_value()) {
        return ret - isn;
    }
    
    sub asn by isn;
    d0 <= asn - checkpoint;
    d1 <= checkpoint - (ans - (1ul << 32));
    return (d0 < d1)? asn : asn - (1ul<<32);
}
```

### For `TCPReceiver`

See my code in [`libsponge/tcp_receiver.cc`](./libsponge/tcp_receiver.cc). It's really easy.

### Remaining Bugs:

**None.**

### Optional: I had unexpected difficulty with: [describe]

The original idea of `unwrap` to add `asn` by `(1ul << 32)` until `asn >= checkpoint` is really slow and only 10 test cases consume 35 seconds. Therefore, I did an optimization to:

```c++
    asn += (checkpoint + isn.raw_value()) / (1ul << 32) * (1ul << 32);
    if (asn < checkpoint + isn.raw_value()) {
        asn += (1ul << 32);
    }
```

Finally it passed the test.
