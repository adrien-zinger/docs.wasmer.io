---
description: >-
  In this example we will learn how to pass data between the host system and
  Wasm using memory pointers.
---

# ► Using memory pointers

Linear memory is a major concept in WebAssembly:

{% hint style="info" %}
Because WebAssembly is sandboxed, memory must be copied between the host \(your Rust application\) and the Wasm module. Upcoming proposals like [WebAssembly Interface Types](https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/Explainer.md) will make this process much easier, but it is still a work in progress.
{% endhint %}

The way that this memory is allocated, freed, passed, organized, etc... can vary depending on the ABI of the Wasm module.

For example, some ABIs will provide functions, either as imports or exports, for allocation and freeing of memory from the host or guest. Some Wasm ABIs may want to control their memory implicitly, for example the ABI may say that the host can reserve memory addresses 0 to 1000 for a special purpose and simply write there directly. You will want to take a look at the documentation of the ABI of your Wasm module, to see what conventions are used for memory allocation.

In this example, let's say we have a Wasm module that can perform transformations on a string passed into the module's memory. This module exports a function that returns a pointer to a fixed size static buffer. This Wasm module will take in a string, and append the string " Wasm is cool!" to the end. This example shows how we can read and write memory from the host \(your Rust application\), and how the Wasm module can also read and write to the same memory.

First we are going to want to initialize a new project. To do this we can navigate to our project folder, or create one. In this example, we will create a new project. Lets create it and navigate to it:

{% tabs %}
{% tab title="Rust" %}
{% hint style="info" %}
The final code for this example can be found on [GitHub](https://github.com/wasmerio/wasmer/blob/master/examples/exports_memory.rs).

_Please take a look at the_ [_setup steps for Rust_](../rust/setup.md)_._
{% endhint %}

```bash
cargo new exported-memory
cd exported-memory
```

We have to modify `Cargo.toml` to add the Wasmer dependencies as shown below:

```yaml
[dependencies]
# The Wasmer API
wasmer = "1.0"
```
{% endtab %}
{% endtabs %}

Now that we have everything set up, let's go ahead and try it out!

{% hint style="info" %}
Before you go further in this example, it is recommended you read the example on the basics of interacting with guest memory:

{% page-ref page="memory.md" %}
{% endhint %}

## Reading the memory

The guest module exports a bunch of things that will be useful for us:

* A memory exported as `mem`;
* A function to retrieve information about the memory contents exported as `load`.

The load function will return two things: the offset of the contents and its length. Let's see how we can use that to read the memory.

{% tabs %}
{% tab title="Rust" %}
```rust
let load = instance
    .exports
    .get_native_function::<(), (WasmPtr<u8, Array>, i32)>("load")?;

let memory = instance.exports.get_memory("mem")?;
```

When importing the function we tell Wasmer that it return a `WasmPtr` as its first return value. This `WasmPtr` is what will give us the ability to interact with the memory.

```rust
let (ptr, length) = load.call()?;
println!("String offset: {:?}", ptr.offset());
println!("String length: {:?}", length);
```
{% endtab %}
{% endtabs %}

Now that we have our pointer, let's read the data. We know we are about to read a string and the `load` function gave us its length. We have everything we need!

{% tabs %}
{% tab title="Rust" %}
```rust
let str = ptr.get_utf8_string(memory, length as u32).unwrap();
println!("Memory contents: {:?}", str);
```
{% endtab %}
{% endtabs %}

Done! We were able to get a string out of the Wasm module memory.

## Writing to the memory

It's now time to write a new string int the guest module's memory. To do that will continue using our pointer.

{% tabs %}
{% tab title="Rust" %}
What we do here is dereferencing the pointer to get a slice of `Cell`s: we want this slice to be the size of the new string starting at the same offset as our pointer. This allows us to completely overwrite the old string with the new one.

{% hint style="info" %}
We could have just changed a single word in the old string. To do that we would have changed the offset and length of the slice when dereferencing the pointer. Here is how we would have done the dereferencing part if we wanted to only change the "World!" part:

```rust
let new_str = b"Wasmer!";
let values = ptr.deref(memory, 7, new_str.len() as u32).unwrap();
```
{% endhint %}
{% endtab %}
{% endtabs %}

## Running

We now have everything we need to run the Wasm module, let's do it!

{% tabs %}
{% tab title="Rust" %}
```text
Compiling module...
Instantiating module...
Memory size (pages) 1 pages
Memory size (bytes) 65536
String offset: 42
String length: 13
Memory contents: "Hello, World!"
New string length: 14
New memory contents: "Hello, Wasmer!"
```

{% hint style="info" %}
If you want to run the examples from the Wasmer [repository](https://github.com/wasmerio/wasmer/) codebase directly, you can also do:

```bash
git clone https://github.com/wasmerio/wasmer.git
cd wasmer
cargo run --example exported-memory --release --features "cranelift"
```
{% endhint %}
{% endtab %}
{% endtabs %}

