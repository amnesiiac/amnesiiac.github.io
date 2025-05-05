---
layout: post
title: "virtio usecase: board rip device emulation (linux, virtualization, driver)"
author: "melon"
date: 1111-10-31 21:44
categories: "2024"
tags:
  - linux
  - virtualization
  - todo
---

$ 1 main.rs:

```text
mod vhu_rip;                                           // module for vhost-user RIP backend functionality

use clap::Parser;                                      // import the clap library for command-line argument parsing
use std::path::PathBuf;                                // import PathBuf for handling file paths
use std::convert::TryFrom;                             // import TryFrom trait for type conversion
use std::io;                                           // import the io module for input/output operations
use vhu_rip::start_backend;                            // import the function to start the backend from the vhu_rip module

type Result<T> = std::result::Result<T, io::Error>;    // def a custom Result type for easier error handling

#[derive(Parser, Debug)]                               // derive parser & debug trait for cmdline arg struct
#[command(author, version, about, long_about = None)]  // metadata for cmdline interface
pub struct RipArgs {
    #[arg(short, long, value_name = "SOCKET")]         // arg for the socket path
    pub socket: String,

    #[arg(short, long, value_name = "FILE")]           // arg for the backend file path
    pub file: String,
}

#[derive(PartialEq, Debug)]                            // derive PartialEq & Debug trait for config struct
struct RipConfiguration {
    socket: PathBuf,                                   // pathBuf for the socket
    file: PathBuf,                                     // pathBuf for the backend file
}

impl TryFrom<RipArgs> for RipConfiguration {           // impl TryFrom trait for converting RipArgs to RipConfiguration
    type Error = io::Error;                            // typedef for Error as io::type ???
    fn try_from(args: RipArgs) -> Result<Self> {       // type Result<T> = std::Result<T, io::Error>
        Ok(RipConfiguration {
            socket: args.socket.into(),                // socket string to PathBuf
            file: args.file.into(),                    // file string to PathBuf
        })
    }
}

fn main() {
    env_logger::init();                                // init logger
    let args = RipArgs::parse();                       // parse cmdline args
    start_backend(args);                               // start backend with parsed args
}
```

$ 2 vhu_rip.rs:

```text
use block::{
    build_serial,                                                            // build serial num for blk dev
    Request,                                                                 // struct for handling blk dev req
    VirtioBlockConfig,                                                       // config struct for virtio blk dev
};
use libc::EFD_NONBLOCK;                                                      // non-blocking eventfd flag
use log::*;
use std::fs::File;
use std::fs::OpenOptions;
use std::io::Read;                                                           // trait for reading from a source
use std::io::{Seek, SeekFrom, Write};                                        // trait for seeking and writing
use std::ops::Deref;                                                         // deref trait for smart pointer deref
use std::ops::DerefMut;                                                      // derefMut trait for mutable deref
use std::os::unix::fs::OpenOptionsExt;                                       // unix-specific extensions for OpenOptions
use std::path::PathBuf;                                                      // path buffer for file paths
use std::process;                                                            // process control
use std::result;                                                             // result type for error handling
use std::sync::atomic::{AtomicBool, Ordering};                               // atomic boolean for thread-safe boolean values
use std::sync::{Arc, Mutex, RwLock, RwLockWriteGuard};                       // sync primitives
use std::vec::Vec;
use std::{convert, error, fmt, io};
use vhost::vhost_user::message::*;                                           // vhost user msg type
use vhost::vhost_user::Listener;                                             // listener for vhost user conn
use vhost_user_backend::{VhostUserBackendMut, VhostUserDaemon, VringRwLock, VringState, VringT}; // vhost user
use virtio_bindings::virtio_blk::*;                                          // virtio blk dev bindings
use virtio_bindings::virtio_config::VIRTIO_F_VERSION_1;                      // virtio version 1 feature flag
use virtio_bindings::virtio_ring::VIRTIO_RING_F_EVENT_IDX;                   // event index feature flag for virtio ring
use virtio_queue::QueueT;                                                    // queue trait for virtio
use vm_memory::GuestAddressSpace;                                            // guest address space for virtual mem
use vm_memory::{bitmap::AtomicBitmap, ByteValued, Bytes, GuestMemoryAtomic}; // mem mgnt types
use vmm_sys_util::{epoll::EventSet, eventfd::EventFd};                       // event handling utilities
use block::raw_file::RawFile;                                                // raw file handling for blk dev
use crate::RipArgs;                                                          // cmdline args struct
use crate::RipConfiguration;                                                 // config struct for ripdev
```

```text
type GuestMemoryMmap = vm_memory::GuestMemoryMmap<AtomicBitmap>;             // alias for guest mem mapped from atomic bitmap

const SECTOR_SHIFT: u8 = 9;                                                  // shift for sector size
const SECTOR_SIZE: u64 = 0x01 << SECTOR_SHIFT;                               // size of a sector in bytes
const BLK_SIZE: u32 = 512;                                                   // block size in bytes

const QUEUE_SIZE: usize = 1024;                                              // max size of queue
const NUM_QUEUES: usize = 1;                                                 // num of queues

trait DiskFile: Read + Seek + Write + Send {}                                // define compound trait for diskfile op
impl<D: Read + Seek + Write + Send> DiskFile for D {}                        // impl DiskFile for any type satisfy the bound

type Result<T> = std::result::Result<T, Error>;                              // alias for Result<T>
type VhostUserBackendResult<T> = std::result::Result<T, std::io::Error>;     // alias for vhost user backend

#[allow(dead_code)]
#[derive(Debug)]
enum Error {                                                      // custom error enum
    CreateKillEventFd(io::Error),                                 // failure when create kill eventfd
    HandleEventNotEpollIn,                                        // failure when handle event (other than input event)
    HandleEventUnknownEvent,                                      // failure when handle unknown event
    PathParameterMissing,                                         // no path provided
    SocketParameterMissing,                                       // no socket provided
}

impl fmt::Display for Error {                                     // impl display trait for Error enum for custom print
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "vdevice_rip_error: {self:?}")                  // format the error as a string
    }
}

impl error::Error for Error {}                                    // clear error trait impl of Error enum

impl convert::From<Error> for io::Error {                         // Error -> io::Error
    fn from(e: Error) -> Self {                                   // impl from will gain into also
        io::Error::new(io::ErrorKind::Other, e)                   // create a new io::Error from the custom error
    }
}
```

```text
struct VhostUserBlkThread {                                       // thread ctx to handle vhost user blk dev operation
    disk_image: Arc<Mutex<dyn DiskFile>>,                         // arc and mutex for thread-safe disk img access
    serial: Vec<u8>,                                              // serial num of the blk dev
    disk_nsectors: u64,                                           // num of sectors in the disk image
    event_idx: bool,                                              // flag for event index support
    kill_evt: EventFd,                                            // eventfd for killing the thread
    writeback: Arc<AtomicBool>,                                   // flag for writeback mode
    mem: GuestMemoryAtomic<GuestMemoryMmap>,                      // guest mem for the thread
}

impl VhostUserBlkThread {                                         // impl struct methods
    fn new(                                                       // ctor
        disk_image: Arc<Mutex<dyn DiskFile>>,                     // disk image to be used
        serial: Vec<u8>,                                          // serial num
        disk_nsectors: u64,                                       // num of sectors in the disk img
        writeback: Arc<AtomicBool>,                               // writeback mode flag
        mem: GuestMemoryAtomic<GuestMemoryMmap>,                  // guest mem
    ) -> Result<Self> {
        Ok(VhostUserBlkThread {
            disk_image,
            serial,
            disk_nsectors,
            event_idx: false,
            kill_evt: EventFd::new(EFD_NONBLOCK).map_err(Error::CreateKillEventFd)?,  // init non-blocking eventfd
            writeback,
            mem,
        })
    }

    fn process_queue(                                                                 // process the req que
        &mut self,
        vring: &mut RwLockWriteGuard<VringState<GuestMemoryAtomic<GuestMemoryMmap>>>, // guard to access the vring
    ) -> bool {
        let mut used_descs = false;                                                   // init clean used_descs
        while let Some(mut desc_chain) = vring                                        // handle available desc chain in vring
            .get_queue_mut()
            .pop_descriptor_chain(self.mem.memory())
        {
            debug!("got an element in the queue");
            let len;                                                                  // len of the processed req
            match Request::parse(&mut desc_chain, None) {                             // parse the req from the desc chain
                Ok(mut request) => {                                                  // if parse succeed, handle the req
                    debug!("element is a valid request");
                    request.set_writeback(self.writeback.load(Ordering::Acquire));    // set writeback mode for the request
                    let status = match request.execute(                               // exec the req
                        &mut self.disk_image.lock().unwrap().deref_mut(),             // lock and deref the disk image
                        self.disk_nsectors,                                           // num of sectors
                        desc_chain.memory(),                                          // mem associated with the desc chain
                        &self.serial,                                                 // serial num
                    ) {
                        Ok(l) => {                                                    // if execution is successful
                            len = l;                                                  // set len to the result
                            VIRTIO_BLK_S_OK                                           // return success status
                        }
                        Err(e) => {                                                   // if execution fails
                            len = 1;                                                  // set len to 1 for error
                            e.status()                                                // return the error status
                        }
                    };
                    desc_chain
                        .memory()
                        .write_obj(status, request.status_addr)                       // write status back to desc chain
                        .unwrap();
                }
                Err(err) => {                                                         // if parsing fails
                    error!("failed to parse available descriptor chain: {:?}", err);  // log the error
                    len = 0;                                                          // set length to 0
                }
            }
            vring.get_queue_mut()
                 .add_used(desc_chain.memory(), desc_chain.head_index(), len)         // add processed desc to used que
                 .unwrap();
            used_descs = true;                                                        // mark the desc as used
        }

        let mut needs_signalling = false;                                             // determine if signaling is needed
        if self.event_idx {                                                           // if event index is enabled
            if vring
                .get_queue_mut()
                .needs_notification(self.mem.memory().deref())                        // check if notification is needed
                .unwrap()
            {
                debug!("signalling queue");                                           // log: signal needed
                needs_signalling = true;                                              // set the flag
            } else {
                debug!("omitting signal (event_idx)");                                // log: signal omitted
            }
        } else {
            debug!("signalling queue");                                               // log: signal needed
            needs_signalling = true;                                                  // set the flag
        }

        if needs_signalling {                                                         // if signal needed
            vring.signal_used_queue().unwrap();                                       // signal the used queue
        }

        used_descs                                                                    // ret the used desc fd
    }
}
```

```text
struct VhostUserBlkBackend {                                          // vhost user blk backend
    threads: Vec<Mutex<VhostUserBlkThread>>,                          // thread for handling blk dev op
    config: VirtioBlockConfig,                                        // config for the virtio blk dev
    queues_per_thread: Vec<u64>,                                      // num of queues per thread
    acked_features: u64,                                              // ack features
    writeback: Arc<AtomicBool>,                                       // writeback mode flag
    mem: GuestMemoryAtomic<GuestMemoryMmap>,                          // guest mem for the backend
}

impl VhostUserBlkBackend {                                            // vhost use blk dev backend
    fn new(                                                           // ctor
        image_path: String,                                           // path to the disk image
        mem: GuestMemoryAtomic<GuestMemoryMmap>,                      // guest memory
    ) -> Result<Self> {
        let mut options = OpenOptions::new();                         // create options for opening the file
        options.read(true);                                           // enable read access
        options.custom_flags(libc::O_DIRECT);                         // set direct I/O flag
        let image: File = options.open(&image_path).unwrap();         // open the disk image file
        let raw_img: RawFile = RawFile::new(image, true);             // create a RawFile from the opened image

        let serial = build_serial(&PathBuf::from(&image_path));       // build the serial number from the image path
        let image = Arc::new(Mutex::new(raw_img)) as Arc<Mutex<dyn DiskFile>>; // wrap the raw image in Arc and Mutex

        let nsectors = (image.lock().unwrap().seek(SeekFrom::End(0)).unwrap()) / SECTOR_SIZE; // num of sectors in disk img
        let config = VirtioBlockConfig {                              // create the virtio blk config
            capacity: nsectors,                                       // set the capacity to the num of sectors
            blk_size: BLK_SIZE,                                       // set the block size
            size_max: 65535,                                          // max size for a single i/o operation
            seg_max: 128 - 2,                                         // max num of segments
            min_io_size: 1,                                           // minimum i/o size
            opt_io_size: 1,                                           // optimal i/o size
            writeback: 1,                                             // enable writeback mode
            ..Default::default()                                      // use default values for other fields
        };

        let mut queues_per_thread = Vec::new();                       // vector to hold the number of queues per thread
        let mut threads = Vec::new();                                 // vector to hold the threads
        let writeback = Arc::new(AtomicBool::new(true));              // initialize writeback mode flag
        for i in 0..NUM_QUEUES {                                      // loop through the number of queues
            let thread = Mutex::new(VhostUserBlkThread::new(          // create a new thread for handling block operations
                image.clone(),                                        // clone the disk image
                serial.clone(),                                       // clone the serial number
                nsectors,                                             // pass the number of sectors
                writeback.clone(),                                    // clone the writeback flag
                mem.clone(),                                          // clone the guest memory
            )?);
            threads.push(thread);                                     // add the thread to the vector
            queues_per_thread.push(0b1 << i);                         // set the queue for the thread
        }

        Ok(VhostUserBlkBackend {                                      // return the new backend instance
            threads,
            config,
            queues_per_thread,
            acked_features: 0,                                        // init ack features as 0
            writeback,
            mem,
        })
    }

    fn update_writeback(&mut self) {                                  // set writeback mode based on ack features
        let writeback =
            if self.acked_features & 1 << VIRTIO_BLK_F_CONFIG_WCE == 1 << VIRTIO_BLK_F_CONFIG_WCE {
                self.config.writeback == 1
            } else {
                self.acked_features & 1 << VIRTIO_BLK_F_FLUSH == 1 << VIRTIO_BLK_F_FLUSH // check if flush feature ack or not
            };

        info!(                                                       // log cache mode change
            "Changing cache mode to {}",
            if writeback {
                "writeback"
            } else {
                "writethrough"
            }
        );
        self.writeback.store(writeback, Ordering::Release);          // store the new writeback mode
    }
}

impl VhostUserBackendMut for VhostUserBlkBackend {                   // impl VhostUserBackendMut trait
    type Bitmap = AtomicBitmap;                                      // alias type for atomic bitmap
    type Vring = VringRwLock<GuestMemoryAtomic<GuestMemoryMmap>>;    // alias type for vring

    fn num_queues(&self) -> usize {                                  // return num of ques
        NUM_QUEUES
    }

    fn max_queue_size(&self) -> usize {                              // return max que size
        QUEUE_SIZE
    }

    fn features(&self) -> u64 {                                      // return available features for the backend
        let avail_features = 1 << VIRTIO_BLK_F_SEG_MAX               // segment maximum feature
            | 1 << VIRTIO_BLK_F_BLK_SIZE                             // block size feature
            | 1 << VIRTIO_BLK_F_FLUSH                                // flush feature
            | 1 << VIRTIO_BLK_F_TOPOLOGY                             // topology feature
            | 1 << VIRTIO_BLK_F_MQ                                   // multi-queue feature
            | 1 << VIRTIO_BLK_F_CONFIG_WCE                           // writeback cache feature
            | 1 << VIRTIO_RING_F_EVENT_IDX                           // event index feature
            | 1 << VIRTIO_F_VERSION_1                                // virtio version feature
            | VhostUserVirtioFeatures::PROTOCOL_FEATURES.bits();     // protocol features

        // avail_features |= 1 << VIRTIO_BLK_F_RO;                   // uncomment to enable read-only feature
        avail_features                                               // return available feature
    }

    fn acked_features(&mut self, features: u64) {                    // ack of the features recv from host
        self.acked_features = features;                              // set ack features
        self.update_writeback();                                     // update the writeback mode
    }

    fn protocol_features(&self) -> VhostUserProtocolFeatures {       // get vhost user proto features
        VhostUserProtocolFeatures::CONFIG                            // config feature
            | VhostUserProtocolFeatures::MQ                          // multi-queue feature
            | VhostUserProtocolFeatures::CONFIGURE_MEM_SLOTS         // mem slots config feature
    }

    fn set_event_idx(&mut self, enabled: bool) {                     // set event idx
        for thread in self.threads.iter() {                          // iterate through all threads
            thread.lock().unwrap().event_idx = enabled;              // set the event index flag
        }
    }

    fn handle_event(                                                 // dev event handler
        &mut self,
        device_event: u16,                                           // device event identifier
        evset: EventSet,                                             // set of events
        vrings: &[VringRwLock<GuestMemoryAtomic<GuestMemoryMmap>>],  // vring references
        thread_id: usize,                                            // thread identifier
    ) -> VhostUserBackendResult<()> {
        if evset != EventSet::IN {                                   // check if the event is an input event
            return Err(Error::HandleEventNotEpollIn.into());         // return error if not
        }

        debug!("event received: {:?}", device_event);                // log the received event

        let mut thread = self.threads[thread_id].lock().unwrap();    // lock the thread for processing
        match device_event {                                         // match on the device event
            0 => {                                                   // handle event with identifier 0
                let mut vring = vrings[0].get_mut();                 // get mutable reference to the vring
                if thread.event_idx {                                // if event index is enabled
                    loop {                                           // loop to process requests
                        vring
                            .get_queue_mut()
                            .enable_notification(self.mem.memory().deref()) // enable notifications for the que
                            .unwrap();
                        if !thread.process_queue(&mut vring) {       // process the que
                            break;                                   // break if no more req
                        }
                    }
                } else {
                    thread.process_queue(&mut vring);                // process the queue without notifications
                }

                Ok(())                                               // ret success
            }
            _ => Err(Error::HandleEventUnknownEvent.into()),         // ret error for unknown events
        }
    }

    fn get_config(&self, _offset: u32, _size: u32) -> Vec<u8> {      // get config
        self.config.as_slice().to_vec()
    }

    fn set_config(&mut self, offset: u32, data: &[u8]) -> result::Result<(), io::Error> {
        let config_slice = self.config.as_mut_slice();               // get mutable slice of the config
        let data_len = data.len() as u32;                            // len of the data to write
        let config_len = config_slice.len() as u32;                  // len of the config
        if offset + data_len > config_len {                          // check for out-of-bounds write
            error!("Failed to write config space");                  // log error
            return Err(io::Error::from_raw_os_error(libc::EINVAL));  // return invalid arg error
        }
        let (_, right) = config_slice.split_at_mut(offset as usize); // split the config slice
        right.copy_from_slice(data);                                 // copy the data into the config
        self.update_writeback();                                     // update the writeback mode
        Ok(())                                                       // return success
    }

    fn exit_event(&self, thread_index: usize) -> Option<EventFd> {   // get exit event for certain thread
        Some(
            self.threads[thread_index]
                .lock()
                .unwrap()
                .kill_evt
                .try_clone()                                         // clone the kill event fd
                .unwrap(),
        )
    }

    fn queues_per_thread(&self) -> Vec<u64> {                        // ret num of que per thread
        self.queues_per_thread.clone()                               // ret a clone of the que per thread
    }

    fn update_memory(                                                // update mem for backend
        &mut self,
        _mem: GuestMemoryAtomic<GuestMemoryMmap>,                    // new guest mem
    ) -> VhostUserBackendResult<()> {
        Ok(())                                                       // return success
    }
}
```

```text
pub fn start_backend(args: RipArgs) {
    let config = RipConfiguration::try_from(args).unwrap();          // parse rip-args as rip-config
    let socket = config.socket.clone();                              // clone socket address ??? what if not use clone
    let file = config.file.clone();                                  // clone file path
    let mem = GuestMemoryAtomic::new(GuestMemoryMmap::new());        // create new guest memory

    let blk_backend = Arc::new(RwLock::new(                          // create new vhost-user block backend
        VhostUserBlkBackend::new(
            file.to_string_lossy().into_owned(),                     // convert file path to string
            mem.clone(),                                             // clone the guest memory
        ).unwrap(),
    ));

    let listener = Listener::new(&socket, true).unwrap();            // create listener
    let name = "vdevice-rip-backend";
    let mut blk_daemon = VhostUserDaemon::new(name.to_string(), blk_backend.clone(), mem).unwrap(); // create blk daemon
    debug!("rip_daemon is created!\n");
    if let Err(e) = blk_daemon.start(listener) {                     // start the daemon with the listener
        error!(
            "Failed to start daemon for vdevice rip with error: {:?}\n",
            e
        );
        process::exit(1);                                            // exit process on failure
    }

    if let Err(e) = blk_daemon.wait() {                              // wait for the daemon to finish
        error!("Error from the main thread: {:?}", e);
    }

    for thread in blk_backend.read().unwrap().threads.iter() {       // iterate through all threads in the backend
        if let Err(e) = thread.lock().unwrap().kill_evt.write(1) {   // write to the kill event to shut down the thread
            error!("Error shutting down worker thread: {:?}", e)     // log any errors that occur
        }
    }
}
```

<hr>

### # Arc::new: a thread-safe way to enable share ownership among multi-threads
`Arc<T>` is a type like `Rc<T>` but it's safe in concurrent situations and safe to share across threads.
it is a atomic ref count type, more about atomic plz ref std::sync::atomic for more details.

why all rust primitive types aren’t atomic and why std type aren’t implemented to use `Arc<T>` by default?  
the thread safety come with performance penalty which only want to pay when it's needed.
for operations on values within a single thread, the `Arc<T>` is not needed to enforce the atomic features.
