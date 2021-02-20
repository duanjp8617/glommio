# A tour of echo.rs example.

1. `let builder = LocalExecutorBuilder::new().pin_to_cpu(1);`  
This is how `LocalExecutorBuilder` gets constructed:
    + ```rust
        LocalExecutorBuilder {
            binding: None,
            spin_before_park: None,
            name: String::from("unnamed"),
            io_memory: 10 << 20,
        }
    + Note the `io_memory` is actually set to 10MB, this seems to be the size of `io_uring` buffers?

