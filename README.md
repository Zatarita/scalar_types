This library was created to assist with parsing endian sensitive content. It allows us to parse the data normally as though it was native endianness. Then when we need the value we can cast the value to the desired endianness.

This saves us from having to create conditional parsing using T::from_xx_bytes, and instead allows us to lazily parse the values. then convert them when we need them. this acts kinda like a endian “superposition”.

Since this also wraps all the basic scalar types, it gives us a common type to pass to functions that tags the data as endian sensitive.

Also included are helper functions to make loading values from a stream quicker.

Let’s look at a basic use case. We have a file that could of been created on a big or little endian system. The implementation of the file specification states the first byte in the file will signal if the rest of the file is big or little endian. Let’s say 0x00 for little endian, and 0x01 for big.

Let’s create a program that parses the first byte and stores the endianness of the file. Then using that endianness cast a value read from the file.
```Rust
use scalar_types::Endian;
use std::io::{BufReader, Result};
  
fn read_some_stuff() -> Result<()> {
    // Binary file contains 01 | 00 00 00 02 .. ..
    //      Big Endian Flag-^    ^-Big Endian 0x2
    // System endianness is little endian
    let file = std::fs::File::open("file.bin")?;
    let mut reader = BufReader::new(file);
 
    // File endianness changes based on system that made it
    // The first byte in the file determines the endianness; 
    // 0 = little, 1 = big
    let mut buffer = vec![0u8];
    reader.read_exact(&mut buffer)?;
 
    // We can then store the endianness as a variable
    // and use this to cast. Allowing us to dynamically
    // cast the data based on the content of the file
    let endianness = {
        if buffer[0] == 0 { 
            Endian::Little(()) 
        } else { 
            Endian::Big(()) 
        }
    };
 
    // Try and read the endian sensitive content normally from stream
    // Endian values are Endian::Native by default
    let endian_value = Endian::<u32>::from_stream(&mut reader);
    let parsed_value = match endian_value {
        Some(value) => value,
        None => panic!("Unable to parse value from stream!")
    };
 
    // Then we convert the value only when we need it.
    // Saving us from having to cast u32::from_xx_bytes for every value
    let expanded_value = match parsed_value.cast(endianness) {
        Some(value) => value,
        None => 0u32
    };
 
    assert_eq!(expanded_value, 2);
    Ok(())
}
```
We can also work the other way around! Let’s create the file we just parsed by using Endian. We could easily use to_le_bytes and to_be_bytes; however, doing so would mean we need to alternate between the two depending on the system we’re targeting. Instead, we can just store the target endianness and use the same code. Dynamically switching the endianness with a stored variable instead. This is a lot cleaner.

Instead of using to_le_bytes or to_be_bytes; we’ll call the to_ne_bytes, and let Endian handle the converting.
```Rust
use scalar_types::Endian;
use std::io::{Result, Write};
use std::fs::File;
use std::path::Path;
fn write_some_stuff() -> Result<()> {
    // Open a file hand for our output
    let mut output = File::create(Path::new("file.bin"))?;
 
    // since we're a little endian system, writing a big endian file
    // we will assign this value explicitly.
    // let's store this for later to use it to cast
    let endianness = Endian::Big(());
 
    // Note that Endian::Native isn't actually aware of the system endianness,
    // it instead acts as an abstract container like a "superposition". 
    // If we wanted to get our native endianness "get_native_endianness()"
    // is a helper function that will return the correct endianness.
    // let endianness = get_native_endianness()?;
 
    // Moving forward:
    // Our spec says we need to write 0x01 for big endian, or 0x00 for little.
    if endianness.is_big() {
        write!(output, "\x01")?;
    } else {
        write!(output, "\x00")?;
    };
 
    // Next we're going to create our native endian 0x02 value
    let endian_value = Endian::new(2u32);
 
    // Finally we write the output
    if let Some(value) = endian_value.cast(endian_value) {
        // We've already handled endianness, so we will use the built-in to_ne_bytes function
        output.write(&value.to_ne_bytes())?;
    }
 
    Ok(())
}
```
This was created to assist me with creating modding tools for video games. As some games share the same data, but the endianness changes based off the console it was built for. Working this way allows me to use the same code for all systems, and cast the values to the native endianness dynamically when needed. I figured I would share it on the off chance other people may find it useful.
