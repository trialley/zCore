## sys_clock_gettime
rCore
```rust
    pub fn sys_clock_gettime(&mut self, clock: usize, mut ts: UserOutPtr<TimeSpec>) -> SysResult {
        info!("clock_gettime: clock: {:?}, ts: {:?}", clock, ts);

        let timespec = TimeSpec::get_epoch();
        ts.write(timespec)?;
        Ok(0)
    }
```

## sys_exec
rCore
```rust
    pub fn sys_exec(
        &mut self,
        path: *const u8,
        argv: *const *const u8,
        envp: *const *const u8,
    ) -> SysResult {
        info!(
            "exec: path: {:?}, argv: {:?}, envp: {:?}",
            path, argv, envp
        );
        let path = check_and_clone_cstr(path)?;
        let args = check_and_clone_cstr_array(argv)?;
        let envs = check_and_clone_cstr_array(envp)?;

        if args.is_empty() {
            error!("exec: args is null");
            return Err(SysError::EINVAL);
        }

        info!("exec: path: {:?}, args: {:?}, envs: {:?}", path, args, envs);

        // Read program file
        let mut proc = self.process();
        let inode = proc.lookup_inode(&path)?;

        // Make new Thread
        // Re-create vm
        let mut vm = self.vm();
        let (entry_addr, ustack_top) =
            Thread::new_user_vm(&inode, args, envs, &mut vm).map_err(|_| SysError::EINVAL)?;

        // Kill other threads
        // TODO: stop and wait until they are finished
        proc.threads.retain(|&tid| tid == self.thread.tid);

        // close file that FD_CLOEXEC is set
        let close_fds = proc
            .files
            .iter()
            .filter_map(|(fd, file_like)| {
                if let FileLike::File(file) = file_like {
                    if file.fd_cloexec {
                        Some(*fd)
                    } else {
                        None
                    }
                } else {
                    None
                }
            })
            .collect::<Vec<_>>();
        for fd in close_fds {
            proc.files.remove(&fd);
        }

        // Activate new page table
        unsafe {
            vm.activate();
        }
        drop(vm);

        // Modify exec path
        proc.exec_path = path.clone();

        // reset disposition (man signal(7))
        for d in proc.dispositions.iter_mut() {
            *d = SignalAction::default();
        }
        drop(proc);

        // Modify the TrapFrame
        self.context.set_ip(entry_addr);
        self.context.set_sp(ustack_top);

        info!("exec:END: path: {:?}", path);
        Ok(0)
    }
```

## sys_read write
```rust
    pub async fn sys_read(
        &mut self, 
        fd: usize, 
        base: UserOutPtr<u8>, 
        len: usize) -> SysResult {

        let mut proc = self.process();

        if !proc.pid.is_init() {
            // we trust pid 0 process
            info!("read: fd: {}, base: {:?}, len: {:#x}", fd, base, len);
        }
        let slice = unsafe { self.vm().check_write_array(base.ptr(), len)? };

        let file_like = proc.get_file_like(fd)?;
        let len = file_like.read(slice).await?;
        Ok(len)
    }

    pub fn sys_write(&mut self, fd: usize, base: *const u8, len: usize) -> SysResult {
        let mut proc = self.process();
        if !proc.pid.is_init() {
            //we trust pid 0 process
            info!("write: fd: {}, base: {:?}, len: {:#x}", fd, base, len);
        }
        let slice = unsafe { self.vm().check_read_array(base, len)? };
        let file_like = proc.get_file_like(fd)?;
        let len = file_like.write(slice)?;
        Ok(len)
    }
```



## sys_poll
```rust
    pub async fn sys_poll(//函数原型：#include <poll.h>  int poll(struct pollfd fd[], nfds_t nfds, int timeout);
        &mut self,
        ufds: UserInOutPtr<PollFd>,
        nfds: usize,
        timeout_msecs: usize,
    ) -> SysResult {
        let proc = self.process();//阻塞获取当前进程
        if !proc.pid.is_init() {//判断进程是否为init进程
            // we trust pid 0 process
            info!(
                "poll: ufds: {:?}, nfds: {}, timeout_msecs: {:#x}",
                ufds, nfds, timeout_msecs
            );
        }

        // check whether the fds is valid and is owned by this process
        let condvars = alloc::vec![&(*TICK_ACTIVITY), &(*SOCKET_ACTIVITY)];

        let polls = ufds.read_array(nfds).unwrap();//拷贝ufgs中的数据到polls数组

        if !proc.pid.is_init() {
            info!("poll: fds: {:?}", polls);//打印所有poll结构体，是一个一个的PollFd
        }

        drop(proc);

        #[must_use = "future does nothing unless polled/`await`-ed"]
        struct PollFuture<'a> {
            polls: Vec<PollFd>,
            syscall: &'a Syscall<'a>,
        }

        impl<'a> Future for PollFuture<'a> {
            type Output = SysResult;

            fn poll(mut self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
                use PollEvents as PE;
                let proc = self.syscall.process();
                let mut events = 0;

                // iterate each poll to check whether it is ready
                for poll in self.as_mut().polls.iter_mut() {//便利检查
                    poll.revents = PE::empty();
                    if let Some(file_like) = proc.files.get(&(poll.fd as usize)) {
                        let mut fut = Box::pin(file_like.async_poll());
                        let status = match fut.as_mut().poll(cx) {
                            Poll::Ready(Ok(ret)) => ret,
                            Poll::Ready(Err(err)) => return Poll::Ready(Err(err)),
                            Poll::Pending => continue,
                        };
                        if status.error {
                            poll.revents |= PE::HUP;
                            events += 1;
                        }
                        if status.read && poll.events.contains(PE::IN) {
                            poll.revents |= PE::IN;
                            events += 1;
                        }
                        if status.write && poll.events.contains(PE::OUT) {
                            poll.revents |= PE::OUT;
                            events += 1;
                        }
                    } else {
                        poll.revents |= PE::ERR;
                        events += 1;
                    }
                }
                drop(proc);

                // some event happens, so evoke the process返回ready，则rust自动唤醒进程
                if events > 0 {
                    return Poll::Ready(Ok(events));
                }

                return Poll::Pending;
            }
        }

        let future = PollFuture {
            polls,
            syscall: self,
        };
        future.await
    }
```