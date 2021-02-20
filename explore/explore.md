# A tour of echo.rs example.

1. `let builder = LocalExecutorBuilder::new().pin_to_cpu(1);`  
    * Below is the `LocalExecutorBuilder` that `LocalExecutorBuilder::new()` returns
        + ```rust
            LocalExecutorBuilder {
                binding: None,
                spin_before_park: None,
                name: String::from("unnamed"),
                io_memory: 10 << 20,
            }
        + Note the `io_memory` is actually set to 10MB, this seems to be the size of `io_uring` buffers?
    * ```rust
        pub fn pin_to_cpu(mut self, cpu: usize) -> LocalExecutorBuilder {
            self.binding = Some(cpu);
            self
        }

2. `builder.name("server").spawn(fut_gen)`
    * `std::thread::Builder::new().name(name).spawn(le_init_closure)`
        * this is what `le_init_closure` looks like
        * `let mut le = LocalExecutor::new(notifier, self.io_memory);`
            * `reactor: Rc::new(parking::Reactor::new(notifier, io_memory)),`
                * `let sys = sys::Reactor::new(notifier, io_memory).expect("..");`
                    *  test memlock_limiit: `let (memlock_limit, _) = Resource::MEMLOCK.get()?;`
                    * create an allocater with 10MB size: `let allocator = Rc::new(UringBufferAllocator::new(io_memory));`
                    * 
        * `le.bind_to_cpu(cpu).unwrap();`
        * `le.queues.borrow_mut().spin_before_park = self.spin_before_park;`
        * `le.init().unwrap();`
            * `self.queues.borrow_mut().available_executors.insert(0, TaskQueue::new(...));`
        * `le.run(async move {fut_gen().await;})`