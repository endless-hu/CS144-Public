Lab 3 Writeup
=============

My name: Zhijing Hu

<del>My SUNet ID: [your sunetid here]</del>

<del>I collaborated with: [list sunetids here]</del>

<del>I would like to thank/reward these classmates for their help: [list sunetids here]</del>

This lab took me about 9 hours to do. <del>I [did/did not] attend the lab session.</del>

### Program Structure and Design of the `TCPSender`:

The core data structure it holds is a map with records the `seqno`, the time since last sent and the `TCPSegment`. 

#### `fill_window()`

It first considers if the sender want to send a `SYN` segment by looking at whether `_next_seqno` is 0 or not. If the sender does need to send a `SYN` segment, call `PushSegment` with `SYN` flag set. Finally stop the function. 

If it does not want to send `SYN`, then make sure the `TCPSender` is at the stage `stream_ongoing`, otherwise terminate the function. 

If it passes the above checks, it is at the stage `stream_ongoing` and ready to send segments until the `_window_size` is filled. 

(A corner case: if the `_window_size` is 0 and we still want to send one more byte, call `PushSegment(1)` and set the `_overflow_byte` to true, so that: **1.** when `tick()` is called, the `_rto` will not be doubled; **2.** when `fill_window()` is called again with `_window_size == 0`, the sender will not send another byte again.)

#### `ack_received(ackno, window_size)`

It first `unwrap` the `ackno`, compare it with the internal `_ackno`. If it finds that the `ackno` is not valid (i.e. smaller than or equal to `_ackno` or bigger than `_next_seqno`), then ignore the `ackno` and stop the function.

Then, it will update the internal `_ackno` and `_window_size`. And it begins to look through the internal map `_segment_in_flight` to find if there is any segment which is fully acknowledged. Evict these segments. If eviction happens, unset the `_overflow_byte`, reset the `_rto` and the `_consecutive_retransmission` counter. 

#### `tick(const size_t ms_since_last_tick)`

It only focus on the first segment in `_segment_in_flight` because we only resend the earliest segment and the segments in `_segment_in_flight` are sorted by their `seqno`. 

First it will check whether the map `_sgment_in_flight` is empty or not. Return immediately if it's empty.

Then add the `ms_since_last_tick` to the time counter. If the counter exceeds the `_rto`, resend the segment and reset the counter. Double `_rto` if the `_overflow_byte` is ==not== set, which means the current segment is not sent when the `_window_size` is 0.

#### `PushSegment(uint16_t size, bool syn)`

Basically it just send a segment with `size` and `syn` bit. But it contains many checks and considers some situations like the bytes are exhausted in `_stream` or `eof` is encountered.

It may be the most complex function among all functions because I did not have a good blue print about how it interacts with `fill_window()` and some checks in it overlap with `fill_window()`. Therefore, it's better to refactor the code. 

### Remaining Bugs:

**None.**

- **Optional: I had unexpected difficulty with: [describe]**

  

- **Optional: I think you could make this lab better by: **

  I did not add any helper functions or classes because I do not have an good idea how to design them to make the program clear and easy to maintain. 

  As a result, my program looks messy and make it hard for further maintenance.

- <del>ptional: I was surprised by: [describe]</del>
- <del>Optional: I'm not sure about: [describe]</del>
