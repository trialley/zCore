# 案例
## sys_clock_gettime
zCore
```rust
    pub fn sys_clock_gettime(&self, clock: usize, mut buf: UserOutPtr<TimeSpec>) -> SysResult {
        info!("clock_gettime: id={:?} buf={:?}", clock, buf);

        let time = timer_now();
        let ts = TimeSpec {
            sec: time.as_secs() as usize,
            nsec: (time.as_nanos() % 1_000_000_000) as usize,
        };
        buf.write(ts)?;

        Ok(0)
    }
```

## sys_exec
zCore
```rust
    pub fn sys_execve(
        &mut self,
        path: UserInPtr<u8>,
        argv: UserInPtr<UserInPtr<u8>>,
        envp: UserInPtr<UserInPtr<u8>>,
    ) -> SysResult {
        info!(
            "execve: path: {:?}, argv: {:?}, envp: {:?}",
            path, argv, envp
        );
        let path = path.read_cstring()?;
        let args = argv.read_cstring_array()?;
        let envs = envp.read_cstring_array()?;
        if args.is_empty() {
            error!("execve: args is null");
            return Err(LxError::EINVAL);
        }

        // TODO: check and kill other threads

        // Read program file
        let proc = self.linux_process();
        let inode = proc.lookup_inode(&path)?;
        let data = inode.read_as_vec()?;

        let vmar = self.zircon_process().vmar();
        vmar.clear()?;
        let loader = LinuxElfLoader {
            syscall_entry: self.syscall_entry,
            stack_pages: 8,
            root_inode: proc.root_inode().clone(),
        };
        let (entry, sp) = loader.load(&vmar, &data, args, envs)?;

        // Modify exec path
        proc.set_execute_path(&path);

        // TODO: use right signal
        self.zircon_process().signal_set(Signal::SIGNALED);

        *self.regs = GeneralRegs::new_fn(entry, sp, 0, 0);
        Ok(0)
    }
```

## sys_read write
```rust
    pub fn sys_read(
        &self, 
        fd: FileDesc, 
        mut base: UserOutPtr<u8>, 
        len: usize) -> SysResult {
        info!("read: fd={:?}, base={:?}, len={:#x}", fd, base, len);
        let proc = self.linux_process();


        let mut buf = vec![0u8; len];
        let file_like = proc.get_file_like(fd)?;
        let len = file_like.read(&mut buf)?;
        base.write_array(&buf[..len])?;
        Ok(len)
    }

    pub fn sys_write(&self, fd: FileDesc, base: UserInPtr<u8>, len: usize) -> SysResult {
        info!("write: fd={:?}, base={:?}, len={:#x}", fd, base, len);
        let proc = self.linux_process();
        let buf = base.read_array(len)?;
        let file_like = proc.get_file_like(fd)?;
        let len = file_like.write(&buf)?;
        Ok(len)
    }
```



## sys_poll
