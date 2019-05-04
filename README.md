# z80
Fast and flexible i8080/Z80 emulator.

[![Build Status](https://travis-ci.org/kosarev/z80.svg?branch=master)](https://travis-ci.org/kosarev/z80)


## Quick facts

* Implements accurate machine cycle-level emulation.

* Supports undocumented instructions, flags and registers.

* Passes the well-known `cputest`, `8080pre`, `8080exer`,
  `8080exm`, `prelim` and `zexall` tests.

* Follows a modular event-driven design for flexible interfacing.

* Employs compile-time polymorphism for zero performance
  overhead.

* Cache-friendly implementation without large code switches and
  data tables.

* Offers default modules for the breakpoints support and generic
  memory.

* Supports multiple independently customized emulator instances.

* Written in strict C++11.

* Does not rely on implementation-defined or unspecified
  behavior.

* Single-header implementation.

* Provides a generic Python 3 API and instruments to create
  custom bindings.

* MIT license.


## Contents

* [Hello world](#hello-world)
* [Adding memory](#adding-memory)
* [Input and output](#input-and-output)
* [Accessing processor's state](#accessing-processors-state)
* [Feedback](#feedback)


## Hello world

```c++
#include "z80.h"

class my_emulator : public z80::z80_cpu<my_emulator> {
public:
    typedef z80::z80_cpu<my_emulator> base;

    my_emulator() {}

    void on_set_pc(z80::fast_u16 pc) {
        std::printf("pc = 0x%04x\n", static_cast<unsigned>(pc));
        base::on_set_pc(pc);
    }
};

int main() {
    my_emulator e;
    e.on_step();
    e.on_step();
    e.on_step();
}
```
[hello.cpp](https://github.com/kosarev/z80/blob/master/examples/hello.cpp)

Building:
```shell
$ git clone git@github.com:kosarev/z80.git
$ cmake z80
$ make
$ make test
$ make hello  # Or 'make examples' to build all examples at once.
```

Running:
```
$ ./examples/hello
pc = 0x0000
pc = 0x0001
pc = 0x0002
```

In this example we derive our custom emulator class,
`my_emulator`, from a
[mix-in](https://en.wikipedia.org/wiki/Mixin) that implements the
logic and default interfaces necessary to emulate the Zilog Z80
processor.
As you may guess, replacing `z80_cpu` with `i8080_cpu` would give
us a similar Intel 8080 emulator.

The `on_set_pc()` method overrides its default counterpart to
print the current value of the `PC` register before changing it.
For this compile-time polymorphism to be able to do its job, we
pass the type of the custom emulator to the processor mix-in as a
parameter.

The `main()` function creates an instance of the emulator and
asks it to execute a few instructions, thus triggering the custom
version of `on_set_pc()`.
The following section reveals what are those instructions and
where the emulator gets them from.


## Adding memory

Every time the CPU emulator needs to access memory, it calls
`on_read()` and `on_write()` methods.
Their default implementations do not really access any memory;
`on_read()` simply returns `0x00`, meaning the emulator in the
example above actually executes a series of `NOP`s, and
`on_write()` does literally nothing.

Since both the reading and writing functions are considered by
the `z80::z80_cpu` class to be handlers, which we know because
they have the `on` preposition in their names, we can use the
same technique as with `on_set_pc()` above to override the
default handlers to actually read and write something.

```c++
class my_emulator : public z80::z80_cpu<my_emulator> {
public:
    ...

    fast_u8 on_read(fast_u16 addr) {
        assert(addr < z80::address_space_size);
        fast_u8 n = memory[addr];
        std::printf("read 0x%02x at 0x%04x\n", static_cast<unsigned>(n),
                    static_cast<unsigned>(addr));
        return n;
    }

    void on_write(fast_u16 addr, fast_u8 n) {
        assert(addr < z80::address_space_size);
        std::printf("write 0x%02x at 0x%04x\n", static_cast<unsigned>(n),
                    static_cast<unsigned>(addr));
        memory[addr] = static_cast<least_u8>(n);
    }

private:
    least_u8 memory[z80::address_space_size] = {
        0x21, 0x34, 0x12,  // ld hl, 0x1234
        0x3e, 0x07,        // ld a, 7
        0x77,              // ld (hl), a
    };
};
```
[adding_memory.cpp](https://github.com/kosarev/z80/blob/master/examples/adding_memory.cpp)

Output:
```
read 0x21 at 0x0000
pc = 0x0001
read 0x34 at 0x0001
read 0x12 at 0x0002
pc = 0x0003
read 0x3e at 0x0003
pc = 0x0004
read 0x07 at 0x0004
pc = 0x0005
read 0x77 at 0x0005
pc = 0x0006
write 0x07 at 0x1234
```


## Input and output

Aside of memory, another major way the processors use to
communicate with the outside world is via input and output ports.
If you read the previous sections, it's now easy to guess that
there is a couple of handlers that do that.
These are `on_input()` and `on_output()`.

Note that the handlers have different types of parameters that
store the port address, because i8080 only supports 256 ports
while Z80 extends that number to 64K.

```c++
    // i8080_cpu
    fast_u8 on_input(fast_u8 port)
    void on_output(fast_u8 port, fast_u8 n)

    // z80_cpu
    fast_u8 on_input(fast_u16 port)
    void on_output(fast_u16 port, fast_u8 n)
```

The example:
```c++
class my_emulator : public z80::z80_cpu<my_emulator> {
public:
    ...

    fast_u8 on_input(fast_u16 port) {
        fast_u8 n = 0xfe;
        std::printf("input 0x%02x from 0x%04x\n", static_cast<unsigned>(n),
                    static_cast<unsigned>(port));
        return n;
    }

    void on_output(fast_u16 port, fast_u8 n) {
        std::printf("output 0x%02x to 0x%04x\n", static_cast<unsigned>(n),
                    static_cast<unsigned>(port));
    }

private:
    least_u8 memory[z80::address_space_size] = {
        0xdb,        // in a, (0xfe)
        0xee, 0x07,  // xor 7
        0xd3,        // out (0xfe), a
    };
};
```
[input_and_output.cpp](https://github.com/kosarev/z80/blob/master/examples/input_and_output.cpp)


## Accessing processor's state

Sometimes it's necessary to examine and/or alter the current
state of the CPU emulator and do that in a way that is
transparent to the custom code in overridden handlers.
For this purpose the default state interface implemented in the
`i8080_state` and `z80_state` classes provdes a number of getters
and setters for registers, register pairs, interrupt flip-flops
and other field constituting the internal state of the emulator.
By convention, calling such functions does not fire up any
handlers. The example below demonstrates a typical usage.

Note that there are no such accessors for memory as it is
external to the processor emulators and they themselves have to
use handlers, namely, the `on_read()` and `on_write()` ones, to
deal with memory.

```c++
class my_emulator : public z80::z80_cpu<my_emulator> {
public:
    ...

    void on_step() {
        std::printf("hl = %04x\n", static_cast<unsigned>(get_hl()));
        base::on_step();

        // Start over on every new instruction.
        set_pc(0x0000);
    }
```
[accessing_state.cpp](https://github.com/kosarev/z80/blob/master/examples/accessing_state.cpp)


## Feedback

Any notes on overall design, improving performance and testing
approaches are highly appreciated. Please use the email given at
<https://github.com/kosarev>. Thanks!
