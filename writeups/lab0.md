Lab 0 Writeup
=============

# Note

### CMake Problem

The given virtual machine has some weird configuration. It DID have g++-8 and clang-6, but if you just run `cmake ..`, it will prompt that the project should be compiled by g++-8 or clang-6(or higher). 

TO fix it, you should instead run:

```
$ CC=clang CXX=clang++ cmake ..
```

### Compile Problem

The project has a bug, so that a normal `cmake ..` will not work when running `make`:

```
/home/cs144/sponge/libsponge/util/parser.cc:36:13: error: shift count >= width of type [-Werror,-Wshift-count-overflow]
        ret <<= 8;
            ^   ~
/home/cs144/sponge/libsponge/util/parser.cc:66:34: note: in instantiation of function template specialization
      'NetParser::_parse_int<unsigned char>' requested here
uint8_t NetParser::u8() { return _parse_int<uint8_t>(); }
                                 ^
1 error generated.
```

To make the project successfully compiled, the `Werror` should be suppressed by setting a global variable:

```
$ export CXXFLAGS="-Wno-error"
```


