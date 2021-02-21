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
                    * create a registry, which is a vector of IOSlice `let registry = vec![IoSlice::new(allocator.as_bytes())];`
                    * with the memory allocator, create the main io_uring: `let mut main_ring = SleepableRing::new(128, "main", allocator.clone())?;`
                        * create a new io_uring with `size` number of entries: `ring: iou::IoUring::new(size as _)?,`
                        * create a submission_queue, which contains the state of the io_uring?: `submission_queue: UringQueueState::with_capacity(size * 4),`
                            * ```rust
                                Rc::new(RefCell::new(UringQueueState {
                                    submissions: VecDeque::with_capacity(cap),
                                    cancellations: VecDeque::new(),
                                }))
                        * other initializations, including setting waiting_submission, name and the memory allocator
                    * with the same memory allocator, create a poll ring: `let poll_ring = PollRing::new(128, allocator.clone())?;`
                        * create a new io_uring with special flags:  `let ring = iou::IoUring::new_with_flags(size as _, iou::SetupFlags::IOPOLL)?;`
                        * create a submission queue: `submission_queue: UringQueueState::with_capacity(size * 4),`
                        * other initializations
                    * if no errors occur, then the following three oprations will be executed in sequence:
                        * `main_ring.registrar().register_buffers(&registry)`
                        * `poll_ring.registrar().register_buffers(&registry)`
                        * `allocator.activate_registered_buffers(0);`
                    * create a latency ring: `let latency_ring = SleepableRing::new(128, "latency", allocator.clone())?;`
                    * obtain the link_fd of the latency ring: `let link_fd = latency_ring.ring_fd();`
                    * create a new link ring fd event source: `let link_rings_src = Source::new(.., link_fd, ..);`
                    * create a event_fd event source: `let eventfd_src = Source::new(.., notifier.eventfd_fd(), ..);`
                    * register the event_fd to the main_ring: `main_ring.install_eventfd(&eventfd_src)`
                        * get a new SubmssionQueueEvent: `let Some(mut sqe) = self.ring.next_sqe()`
                        * increase the waiting_submission count: `self.waiting_submission += 1;`
                        * `let op = UringDescriptor {..};`
                            * record the event_fd: `fd: eventfd_src.raw(),`
                            * `add_source(eventfd_src, self.submission_queue.clone())`
                                * clone the source: `let item = source.inner.clone();`
                                * insert the source into a thread local source map, and acquire the index of the location in the map: `let id = x.borrow_mut().alloc(item);`
                                * record the id on the source map and the registered submission queue in the source: `source.inner.enqueued.set(Some(EnqueuedSource {id,queue: queue.clone(),}));`
                                * return the id in the source map
                            * add the index by 1 and store to user_data: `user_data: to_user_data(add_source(eventfd_src, self.submission_queue.clone())),`
                            * `args: UringOpDescriptor::Read(0, 8),`
                        * clone the memory allocator: `let allocator = self.allocator.clone();`
                        * 
                    
        * `le.bind_to_cpu(cpu).unwrap();`
        * `le.queues.borrow_mut().spin_before_park = self.spin_before_park;`
        * `le.init().unwrap();`
            * `self.queues.borrow_mut().available_executors.insert(0, TaskQueue::new(...));`
        * `le.run(async move {fut_gen().await;})`