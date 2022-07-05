Lab 1 Writeup
=============

My name: Zhijing Hu

This lab took me about 8 hours to do. 

### Program Structure and Design of the `StreamReassembler`:

Since segments may come in arbitrary order,  The `StreamReassembler` should keep the order of the incoming string. I chose `map` to implement this data structure. The `map` member in `StreamReassembler` maps the index of a string to a pair which contains the string and its `eof` sign. I also kept `_unassembled_size` and `_first_unassembled` to keep track of the reassemble process. 

When a substring comes in, my program will 

1. Check if it can be pushed into the stream(its index is behind the `_first_unassembled`)

​      1.1. if it is qualified, truncate the first few bytes which may overlapped with the already reassembled stream and push it into stream*(No need to check overflow because the stream `_output` will automatically abort the overflowed part)*;

​       1.2. Then check the `_wait_list` to see if there's any string can be pushed into stream

2. if it cannot be assembled, then push it into `_wait_list`: 

   ​    2.1.1.1. If the index not existed in `_wait_list`, simply insert the `{index, {data, eof}}` into `_wait_list`;

   ​    2.1.1.2. Merge it with its previous string in the `_wait_list`.

   2.1.2. else, there was a string with the same index as the incoming string, compare their length - if the incoming string is shorter, there's no need to update the existed string; Otherwise concatenate them together and update `_unassembled_size`.

   2.2. After handling the incoming string, now we try merging the newly inserted(or updated) string with its higher-indexed string until no string can be merged.

   2.3. Finally, check whether the total bytes in `_output` and `_wait_list` exceeds the capacity. If  they do, truncate the last few bytes in `_wait_list` until it fits in capacity

To get more details, see the source code:

- [libsponge/stream_reassembler.hh](../libsponge/stream_reassembler.hh)
- [libsponge/stream_reassembler.cc](../libsponge/stream_reassembler.cc)

### Remaining Bugs:
None

### Optional: I had unexpected difficulty with: 

The design of `push_substring` method. 

At first, I thought the problem was quite easy, and I decided to implement it without any helper function. However, as the lab progress, I found that there are too many situations to consider and my implementation code became longer and longer, which made it hard for me to debug. 

This failed version wasted me around 5 hours :(

The final code was in the Appendix. It was really ugly :( , and it had many bugs. I attach this failed code to remind me the importance of helper function and the elegant design.

### Appendix

The ***FAILED*** code: 

```c++
#include "stream_reassembler.hh"

#include <iostream>

// Dummy implementation of a stream reassembler.

// For Lab 1, please replace with a real implementation that passes the
// automated checks run by `make check_lab1`.

// You will need to add private members to the class declaration in `stream_reassembler.hh`

template <typename... Targs>
void DUMMY_CODE(Targs &&.../* unused */) {}

using namespace std;

StreamReassembler::StreamReassembler(const size_t capacity) : _wait_list(), _output(capacity), _capacity(capacity) {}

//! \details This function accepts a substring (aka a segment) of bytes,
//! possibly out-of-order, from the logical stream, and assembles any newly
//! contiguous substrings and writes them into the output stream in order.
void StreamReassembler::push_substring(const string &data, const size_t index, const bool eof) {
    if (!_output.error()) {
        // cout << "\n Isempty: " << _wait_list.empty() << endl;
        if (index <= _first_unassembled) {                    // the substring conforms its order, so push it to output
            if (_first_unassembled - index >= data.size()) {  // the substring is totally overlapped
                if (eof && _first_unassembled - index == data.size()) {
                    // The substring is empty, but has a EOF signal
                    _output.end_input();
                }
                // ignore the overlapped substring
                return;
            }
            _first_unassembled += _output.write(data.substr(_first_unassembled - index, data.size()));  // strip overlap
            if (eof && _capacity >= _output.buffer_size() + _unassembled_size) {
                // no need to perform following check, because it is
                // the last substring and it is merged into stream
                _output.end_input();
                return;
            }
            // After reassambling, check whether there's a
            // substring which can be reassambled in wait list
            map<size_t, pair<string, bool>>::iterator to_erase = _wait_list.lower_bound(0);
            // cout << (to_erase == _wait_list.end()) << " " << _wait_list.empty() << endl;

            while (to_erase != _wait_list.end() && to_erase->first <= _first_unassembled) {
                if (_first_unassembled - to_erase->first >= to_erase->second.first.size()) {
                    // The whole string is overlapped, ignore and erase it
                    _unassembled_size -= to_erase->second.first.size();
                    _wait_list.erase(to_erase);
                    to_erase = _wait_list.lower_bound(0);
                    continue;
                }
                // truncate overlapping part
                string trunc =
                    to_erase->second.first.substr(_first_unassembled - to_erase->first, to_erase->second.first.size());
                // push the substring into stream
                size_t written_bytes = _output.write(trunc);
                _unassembled_size -= to_erase->second.first.size();
                _first_unassembled += written_bytes;
                // check if the pushed string overflows
                if (written_bytes < trunc.size()) {
                    // the pushed string overflows, add the overflowed part to unaccepted
                    string unaccepted = trunc.substr(written_bytes, trunc.size());
                    _wait_list.insert({_first_unassembled, {unaccepted, eof}});
                    _wait_list.erase(to_erase);
                    // cannot continue merging because it reaches its capacity
                    return;
                } else if (to_erase->second.second &&  // if the merged substring has eof signal
                           _capacity >= _output.buffer_size() + _unassembled_size) {
                    _output.end_input();  // end the input
                    // no need to perform check, because it is
                    // the last substring and it is merged into stream
                    _wait_list.erase(to_erase);
                    return;
                }
                _wait_list.erase(to_erase);
                to_erase = _wait_list.lower_bound(0);
            }
        } else {  // we should put it in wait list
                  // first check if the substring is overlapped with other lower-indexed strings in the wait list
            map<size_t, pair<string, bool>>::iterator next_lower_string = _wait_list.upper_bound(index);

            // special case: next_lower_string cannot decrement because:
            // 1. current string is highest-indexed;
            // 2. next_lower_string is at beginning.
            if (next_lower_string == _wait_list.begin() && next_lower_string->first > index) {
                if (index + data.size() < next_lower_string->first) {
                    // not overlap, insert it
                    // we should ensure before insertion that adding the substring
                    // does not exceed capacity
                    if (data.size() + _unassembled_size + _output.buffer_size() > _capacity) {
                        if (_unassembled_size + _output.buffer_size() == _capacity) {
                            // if current reassambler is full, the whole substring is unaccepted
                            _wait_list.insert({index, {data, eof}});
                            return;
                        } else {
                            // we split the string into unassembled part and unaccepted part
                            string unassemabled = data.substr(0, _capacity - _unassembled_size - _output.buffer_size());
                            string unaccepted =
                                data.substr(_capacity - _unassembled_size - _output.buffer_size(), data.size());
                            _wait_list.insert({index, {unassemabled, false}});
                            _unassembled_size += unassemabled.size();
                            _wait_list.insert({index + unassemabled.size(), {unaccepted, eof}});
                            return;
                        }
                    }
                    // not exceed capacity, safe to insert
                    _wait_list.insert({index, {data, eof}});
                    _unassembled_size += data.size();
                    return;
                } else {
                    // overlap, merge them
                    size_t overlapped_len = index + data.size() - next_lower_string->first;
                    string merged;
                    if (overlapped_len < next_lower_string->second.first.size()) {
                        merged = data + next_lower_string->second.first.substr(overlapped_len,
                                                                               next_lower_string->second.first.size());
                        _unassembled_size += (data.size() - overlapped_len);
                    } else {
                        merged = data;
                        _unassembled_size += (data.size() - next_lower_string->second.first.size());
                    }
                    _wait_list.insert({index, {merged, eof}});
                    _wait_list.erase(next_lower_string);
                    // try merging higher-indexed segments
                    map<size_t, pair<string, bool>>::iterator to_erase = _wait_list.upper_bound(index);
                    // while next_lower_string overlaps with its next higher string, merge them
                    while (to_erase->first <= next_lower_string->first + next_lower_string->second.first.size() &&
                           to_erase != _wait_list.end()) {
                        if (next_lower_string->first + next_lower_string->second.first.size() >= to_erase->first) {
                            overlapped_len =
                                next_lower_string->first + next_lower_string->second.first.size() - to_erase->first;
                            // cout << "\n" << next_lower_string->second.first << "\noverlap_len: " << overlapped_len <<
                            // endl;
                            next_lower_string->second.first +=
                                to_erase->second.first.substr(overlapped_len, to_erase->second.first.size());
                            _unassembled_size -= (to_erase->second.first.size() - overlapped_len);
                            next_lower_string->second.second = to_erase->second.second;
                            // cout << "\n" << next_lower_string->second.first << "\noverlap_len: " << overlapped_len <<
                            // endl;
                        }  // else ignore the higher-indexed string
                        _wait_list.erase(to_erase);
                        to_erase = _wait_list.upper_bound(index);
                    }
                    return;
                }
            }
            if (next_lower_string == _wait_list.end()) {
                if (_wait_list.empty()) {
                    _wait_list.insert({index, {data, eof}});
                    _unassembled_size += data.size();
                    return;
                } else {
                    // since the current string will be at the new highest pos,
                    // first check if it is overlapped with the current highest string
                    map<size_t, pair<string, bool>>::reverse_iterator highest_string = _wait_list.rbegin();
                    if (highest_string->first + highest_string->second.first.size() < index) {
                        // not overlap
                        _wait_list.insert({index, {data, eof}});
                        _unassembled_size += data.size();
                        return;
                    } else {  // overlapped, we should merge it with highest_string
                        size_t overlapped_len = highest_string->first + highest_string->second.first.size() - index;
                        if (overlapped_len < data.size()) {
                            highest_string->second.first += data.substr(overlapped_len, data.size());
                            _unassembled_size += (data.size() - overlapped_len);
                        }  // else ignore it
                        return;
                    }
                }
            }

            // normal case: next_lower_string exists and can decrement
            next_lower_string--;
            if (next_lower_string->first + next_lower_string->second.first.size() < index) {
                // The string is NOT overlapped with other strings

                // second, we should ensure that adding the substring
                // does not exceed capacity
                if (data.size() + _unassembled_size + _output.buffer_size() > _capacity) {
                    if (_unassembled_size + _output.buffer_size() == _capacity) {
                        // if current reassambler is full, the whole substring is unaccepted
                        _wait_list.insert({index, {data, eof}});
                        return;
                    } else {
                        // we split the string into unassembled part and unaccepted part
                        string unassemabled = data.substr(0, _capacity - _unassembled_size - _output.buffer_size());
                        string unaccepted =
                            data.substr(_capacity - _unassembled_size - _output.buffer_size(), data.size());
                        _wait_list.insert({index, {unassemabled, false}});
                        _unassembled_size += unassemabled.size();
                        _wait_list.insert({index + unassemabled.size(), {unaccepted, eof}});
                        return;
                    }
                }
                _wait_list.insert({index, {data, eof}});
                _unassembled_size += data.size();
            } else {  // The string IS overlapped with other strings
                size_t overlapped_len;
                if (next_lower_string->first + next_lower_string->second.first.size() >= index) {
                    // the current substring can be merged with the previous one
                    overlapped_len = next_lower_string->first + next_lower_string->second.first.size() - index;
                    _unassembled_size += (data.size() - overlapped_len);
                    next_lower_string->second.first += data.substr(overlapped_len, data.size());
                    next_lower_string->second.second = eof;
                } else {  // else ignore it
                    return;
                }

                // try merging higher-indexed segments
                map<size_t, pair<string, bool>>::iterator to_erase = _wait_list.upper_bound(index);
                // while next_lower_string overlaps with its next higher string, merge them
                while (to_erase->first <= next_lower_string->first + next_lower_string->second.first.size() &&
                       to_erase != _wait_list.end()) {
                    if (next_lower_string->first + next_lower_string->second.first.size() >= to_erase->first) {
                        overlapped_len =
                            next_lower_string->first + next_lower_string->second.first.size() - to_erase->first;
                        // cout << "\n" << next_lower_string->second.first << "\noverlap_len: " << overlapped_len <<
                        // endl;
                        if (overlapped_len < to_erase->second.first.size()) {
                            next_lower_string->second.first +=
                                to_erase->second.first.substr(overlapped_len, to_erase->second.first.size());
                            _unassembled_size -= (to_erase->second.first.size() - overlapped_len);
                        } else {
                            _unassembled_size -= to_erase->second.first.size();
                        }
                        next_lower_string->second.second = to_erase->second.second;
                        // cout << "\n" << next_lower_string->second.first << "\noverlap_len: " << overlapped_len <<
                        // endl;
                    }  // else ignore the higher-indexed string
                    _wait_list.erase(to_erase);
                    to_erase = _wait_list.upper_bound(index);
                }
                // cout << "\n"
                //<< next_lower_string->second.first << "\noverlap_len: " << overlapped_len
                //<< " isEmpty: " << _wait_list.empty() << endl;
            }
        }
    }
}

size_t StreamReassembler::unassembled_bytes() const { return _unassembled_size; }

bool StreamReassembler::empty() const { return _wait_list.empty(); }

```

