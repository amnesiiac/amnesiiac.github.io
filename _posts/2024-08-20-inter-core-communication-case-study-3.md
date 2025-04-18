---
layout: post
title: "inter-core communication on amp arch III (simulation) (rust, icc, amp)"
author: "melon"
date: 1111-08-19 20:40
categories: "2024"
tags:
  - rust
  - virtualization
  - driver
  - icc
  - async
---

this article focus on illustration of the rust app, which working as a proxy of the rpmsg based on core 0
s6-service scripts.

<hr>

### # code walk through inco-proxy app (rust)
1 rpmsg.rs:

```text
use crate::inotify::REDUN_NOTIFY;                                                     // import pub constant
use async_trait::async_trait;                                                         // import macro async_trait
use pnet_datalink::{self, Channel, DataLinkReceiver, DataLinkSender, MacAddr, NetworkInterface};
use pnet_packet::ethernet::{EtherType, EthernetPacket, MutableEthernetPacket};
use pnet_packet::Packet;
use std::io;
use std::str;
use std::sync::mpsc::{Receiver, Sender};                                              // multi producer single consumer ch
use tokio::time::{sleep, Duration};

const ETHERNET_HEADER_LEN: usize = 14;

#[async_trait]
pub trait Services {                                                                  // async trait def
    async fn watcher(&mut self, sync_tx: Sender<String>);                             // watcher
    async fn handler(&mut self, sync_rx: Receiver<String>);                           // handler
}

pub struct RawSocket {
    pub interface: NetworkInterface,                                                  // net itf associated with rawsocket
    pub receiver: Box<dyn DataLinkReceiver>,                                          // receiver with type DataLinkReceiver
    pub sender: Box<dyn DataLinkSender>,                                              // sender with type DataLinkSender
}

impl RawSocket {                                                                      // impl specific method for rawsocket
    pub fn new(name: &str) -> Self {                                                  // rawsocket ctor
        let interface_names_match = |iface: &NetworkInterface| iface.name == name;    // define closure to match itf name
        let interfaces = pnet_datalink::interfaces();                                 // get all itf
        let interface = interfaces                                                    // get itf by name
            .into_iter()
            .filter(interface_names_match)
            .next()
            .unwrap();                                                                // ?

        let (tx, rx) = match pnet_datalink::channel(&interface, Default::default()) { // init tx,rx ch on itf by pnet
            Ok(Channel::Ethernet(tx, rx)) => (tx, rx),                                // ch is ethernet ch
            Ok(_) => panic!("Unhandled channel type"),                                // panic for other ch types
            Err(e) => panic!(                                                         // panic for error
                "An error occurred when creating the datalink channel: {}",
                e
            ),
        };

        RawSocket {                                                                   // return rawsocket obj for ctor
            interface: interface,
            receiver: rx,
            sender: tx,
        }
    }

    pub fn send(&mut self, buf: &str) -> Option<io::Result<()>> {               // impl rawsocket.send
        let len = buf.len() + ETHERNET_HEADER_LEN;                              // pkt len = ethernet hdr len + data buf len
        println!("[rpmsg] send raw packet to {:?}", self.interface.name);
        return self.sender.build_and_send(1, len, &mut |packet| {               // invoke backend sender obj for pkt sending
            let mut packet = MutableEthernetPacket::new(packet).unwrap();
            packet.set_destination(MacAddr::broadcast());                       // set rawsocket dst mac addr as broadcast
            packet.set_source(self.interface.mac.unwrap());                     // set src mac addr
            packet.set_ethertype(EtherType(0x8800));                            // set rawsocket ethernet header type
            packet.set_payload(buf.as_bytes().to_vec().as_slice());             // set payload by databuf
        });
    }
}

#[async_trait]
impl Services for RawSocket {                                                   // impl trait itf for rawsocket
    async fn watcher(&mut self, sync_tx: Sender<String>) {                      // async fn for non-block pkt watcher
        let mut error_printed = false;
        loop {
            match self.receiver.next() {                                        // wait for next pkt from recv ch (Result)
                Ok(packet) => {                                                 // if match ret Result as packet
                    let packet = EthernetPacket::new(packet).unwrap();          // init ethernet pkt
                    let payload = packet.payload().clone();                     // clone payload from the pkt

                    let dst = str::from_utf8(payload).expect("Invalid utf-8");  // convert payload encoding
                    sync_tx.send(dst.to_string()).unwrap();                     // send the payload to sync_tx ch
                    if error_printed {
                        eprintln!("Recovered from error");
                        error_printed = false;
                    }
                }
                Err(e) => {                                                     // err handling
                    if !error_printed {
                        eprintln!("An error occurred while reading: {}", e);
                        error_printed = true;
                    }
                    continue;
                }
            };
            sleep(Duration::from_millis(100)).await;                            // sleep & await to giveup cpu
        }
    }

    async fn handler(&mut self, sync_rx: Receiver<String>) {                    // async pkt handler
        loop {
            if let std::result::Result::Ok(dst) = sync_rx.try_recv() {          // if recv msg from sync_rx ch
                match dst.as_str() {
                    REDUN_NOTIFY => self.send(REDUN_NOTIFY),                    // if REDUN_NOTIFY, invoke self.send
                    _ => None,                                                  // do nothing for other cases
                };
            }
            sleep(Duration::from_millis(100)).await;                            // sleep & await to giveup cpu
        }
    }
}
```

2 inotify.rs:

```text
use crate::rpmsg::Services;                                                       // import Services trait
use async_trait::async_trait;                                                     // import macro async_trait
use inotify::Inotify;                                                             // for fs event monitoring
use std::fs;
use std::fs::File;
use std::fs::OpenOptions;
use std::io;
use std::io::Write;
use std::path::Path;
use std::sync::mpsc::{Receiver, Sender};                                          // import mpsc receiver & sender ch
use tokio::time::{sleep, Duration};                                               // time & duration from tokio runtime

pub const REDUN_NOTIFY: &str = "redun_notify_irq";
pub const IHUB_HWWD: &str = "ihub_hwwd_irq";
pub const IHUB_RESET: &str = "ihub_reset_irq";

pub struct IrqDevs {                                                              // abstract irqdev whose irq to monitor
    pub redun_notify_path: String,                                                // dev path for redun swo notify
    pub ihub_hwwd_path: String,                                                   // dev path for ihub watchdog
    pub ihub_reset_path: String,                                                  // dev path for ihub sw reset file
}

impl IrqDevs {                                                                    // impl irqdev methods
    pub fn new(path: &str) -> Self {                                              // irqdev ctor 
        let data = fs::read_to_string(path).expect("Unable to read file");
        let json: serde_json::Value = serde_json::from_str(&data).expect("JSON does not have correct format.");

        let ihub_irq_dev = json                                                   // get file.ihub-irq-dev.path field
            .get("file").unwrap()
            .get("ihub-irq-dev").unwrap()
            .get("path").unwrap()
            .to_string()
            .replace("\"", "");                                                   // remove quote from string
        let ihub_irq_dev_path = Path::new(ihub_irq_dev.as_str());                 // init path by str
        fs::create_dir_all(ihub_irq_dev_path.join("")).unwrap();                  // create dev path
        let ihub_config_dev_regs = json                                           // get misc.ihub_config_dev.regs field
            .get("misc").unwrap()
            .get("ihub_config_dev").unwrap()
            .get("regs").unwrap();

        let mut redun_notify_trig = "message_6_trig".to_string();                 // redun swo notify str to ihub
        match ihub_config_dev_regs.get("redun_notify_trig") {                     // check if redun_notify_trig is defined
            Some(reg) => {                                                        // get redun swo notify path
                let reg_value = reg.get("reg").unwrap().to_string();
                let split: Vec<&str> = reg_value.split(".").collect();
                redun_notify_trig = split[1].to_string().replace("\"", "");
            }
            None => println!("redun_notify_trig use default path"),               // log if use default path
        }
        let redun_notify_trig_path = ihub_irq_dev_path.join(redun_notify_trig);
        File::create(redun_notify_trig_path.clone())
            .expect("redun_notify_trig file create failed");

        let mut ihub_hwwd_irq = "message_5_state".to_string();                    // hw watchdog irq msg str
        match ihub_config_dev_regs.get("ihub_hwwd_irq") {                         // check if ihub_hwwd_irq is defined
            Some(reg) => {
                let reg_value = reg.get("reg").unwrap().to_string();
                let split: Vec<&str> = reg_value.split(".").collect();
                ihub_hwwd_irq = split[1].to_string().replace("\"", "");
            }
            None => println!("ihub_hwwd_irq use default path"),                   // log if use the default path
        }
        let ihub_hwwd_irq_path = ihub_irq_dev_path.join(ihub_hwwd_irq);
        File::create(ihub_hwwd_irq_path.clone()).expect("ihub_hwwd_irq file create failed");

        let mut ihub_reset_irq = "message_4_state".to_string();                        // ihub hw reset msg str
        match ihub_config_dev_regs.get("ihub_reset_irq") {                             // check if ihub_reset_irq is defined
            Some(reg) => {
                let reg_value = reg.get("reg").unwrap().to_string();
                let split: Vec<&str> = reg_value.split(".").collect();
                ihub_reset_irq = split[1].to_string().replace("\"", "");
            }
            None => println!("ihub_reset_irq use default path"),                       // log if use the default path
        }
        let ihub_reset_irq_path = ihub_irq_dev_path.join(ihub_reset_irq);
        File::create(ihub_reset_irq_path.clone()).expect("ihub_reset_irq file create failed");

        IrqDevs {                                                                      // return irqdev obj
            redun_notify_path: redun_notify_trig_path.as_path().display().to_string(), // set redun swo notify path
            ihub_hwwd_path: ihub_hwwd_irq_path.as_path().display().to_string(),        // set ihub watchdog path
            ihub_reset_path: ihub_reset_irq_path.as_path().display().to_string(),      // set ihub reset path
        }
    }

    fn write(&mut self, path: String, buf: &str) -> Option<io::Result<()>> {       // irqdev write method
        println!("[inotify] write file: {:?}", path);
        let mut f = OpenOptions::new()                                             // open file with opt set
            .append(false)                                                         // append to file forbidden
            .create(true)                                                          // create if not exist
            .write(true)                                                           // write auth
            .open(path)                                                            // open the file
            .expect("Unable to open file");                                        // handle err if any
        f.write_all(buf.as_bytes()).expect("Unable to write data");                // write buf to file with err handling

        Some(Ok(()))                                                               // return ok
    }
}

#[async_trait]
impl Services for IrqDevs {                                                        // import service trait for irqdev
    async fn watcher(&mut self, sync_tx: Sender<String>) {                         // async fn to watch file event
        let mut inotify = Inotify::init().expect("Failed to initialize inotify");  // init inotify instance
        let redun_notify_descriptor = inotify                                      // add file watch to inotify instance
            .add_watch(
                self.redun_notify_path.clone(),                                    // watch redun status file (close & write)
                inotify::WatchMask::CLOSE_WRITE,
            )
            .expect("Failed to add inotify watch for notify file");

        let mut buffer = [0; 32];                                                  // buffer to hold inotify events
        loop {                                                                     // infinite loop for event 
            let events = inotify                                                   // block reading events from inotify obj
                .read_events_blocking(&mut buffer)
                .expect("Failed to read inotify events");
            for event in events {                                                  // iterate over captured event by inotify
                if event.wd == redun_notify_descriptor {                           // if event wd is redun notify fd
                    println!("[inotify] redun_notify file modified: {:?}", event.name);
                    sync_tx.send(REDUN_NOTIFY.to_string()).unwrap();               // send a redun notify str by sync ch
                }
            }
            sleep(Duration::from_millis(100)).await;                               // sleep & await to giveup cpu
        }
    }

    async fn handler(&mut self, sync_rx: Receiver<String>) {                       // async fn to handle msg
        loop {                                                                     // infinite loop to check for msg
            if let std::result::Result::Ok(dst) = sync_rx.try_recv() {             // try recv msg from sync ch 
                match dst.as_str() {
                    IHUB_HWWD => self.write(self.ihub_hwwd_path.clone(), "1"),     // IHUB_HWWD -> write 1 to watchdog path
                    IHUB_RESET => self.write(self.ihub_reset_path.clone(), "1"),   // IHUB_RESET -> write 1 to reset path
                    _ => None,                                                     // ignore other msg
                };
            }
            sleep(Duration::from_millis(100)).await;                               // sleep & await to giveup cpu
        }
    }
}
```

3 main.rs:

```text
use rpmsg::Services;                                           // import rpmsg
use std::env;                                                  // for access env var & cmdline arg
use std::sync::mpsc;                                           // multi producer single consumer msg ch

mod inotify;                                                   // introduce inotify.rs content as mod
mod rpmsg;                                                     // introduce rpmsg.rs content as mod

#[tokio::main]                                                 // macro tokio: mark main as async fn, startup tokio runtime,
async fn main() {                                              // allow await in it, allow async spawn task inside
    let itf_name = env::args().nth(1).expect("Invalid interface name");
    let udrv_json = env::var("UDRV_JSON_PATH").expect("Invalid udrv.json path");

    let (inotify_tx, inotify_rx) = mpsc::channel::<String>();  // setup mpsc ch for str msg communication
    let (rawsock_tx, rawsock_rx) = mpsc::channel::<String>();

    let itf_name_c = itf_name.clone();                         // clone var, avoid ownership issue in later async ctx
    let udrv_json_c = udrv_json.clone();

    tokio::spawn(async move {                                  // spawn async task run concurrently
        rpmsg::RawSocket::new(itf_name.as_str())               // create new rawsocket obj with itf name assigned
            .watcher(rawsock_tx)                               // setup watcher for send msg to tx ch
            .await;
    });

    tokio::spawn(async move {
        rpmsg::RawSocket::new(itf_name_c.as_str())
            .handler(inotify_rx)                               // setup handler for recv msg from rx ch
            .await;
    });

    tokio::spawn(async move {
        inotify::IrqDevs::new(udrv_json.as_str())              // create new irqdev obj
            .watcher(inotify_tx)                               // setup watcher for send irq msg to tx ch
            .await;
    });

    inotify::IrqDevs::new(udrv_json_c.as_str())                // create new irqdev obj
        .handler(rawsock_rx)                                   // setup handler to incoming rx msg
        .await;
}
```

4 cargo.toml:

```text
[package]
name = "inco_proxy"
version = "0.1.3"
edition = "2024"
authors = ["the inco_proxy app developers"]

[dependencies]
pnet_packet = "0.31.0"
pnet_datalink = "0.31.0"
inotify = "0.9"
serde_json = "1.0.66"
async-trait = "0.1.57"
tokio = {version = "1.4", features = ["full"]}

[profile.release]
strip = true
```

5 app usages (startup by s6 init-system on nt board):

```text
$ UDRV_JSON_PATH=/isam/user/plat_srv/udrv.json start_app --bg /isam/user/host/apps/inco_proxy rpmsg4000 > /tmp/messages
```

6 udrv.json snippet highlights:

```text
{
    "configs" : {
        "print_level": "error"
    },

    ...

    "file": {
        "compatible": "udrv file controller",

        "eps-host": {
            "compatible": "udrv regular file device",
            "path": "/isam/user/host/chassis/eps/"
        },
        ...
        "ihub-irq-dev": {
            "compatible": "udrv regular file device",
            "path": "/isam/user/plat_srv/devdata/dev"
        },
    },

    ...

    "misc": {
        "compatible": "udrv misc controller",
        ...
        "ihub_config_dev": {
            "compatible": "udrv field supported device",
            "regs": {
                "ihub_cmdline":             {"reg": "ihub-cmdline.ihub-cmd",            "privatedata": "0x0"},
                "ihub_uart_irqfwd_msgdata": {"reg": "ihub-irq-dev.uart_irqfwd_msgdata", "privatedata": "0x0"},
                "ihub_uart_irqfwd_state":   {"reg": "ihub-irq-dev.uart_irqfwd_state",   "privatedata": "0x0"},
                "ihub_hwwd_irq":            {"reg": "ihub-irq-dev.message_5_state",     "privatedata": "0x0"},
                "ihub_reset_irq":           {"reg": "ihub-irq-dev.message_4_state",     "privatedata": "0x0"},
                "redun_notify_irq":         {"reg": "ihub-irq-dev.message_6_state",     "privatedata": "0x0"},
                "redun_notify_trig":        {"reg": "ihub-irq-dev.message_6_trig",      "privatedata": "0x0"},
                "ihub_hwwd_irq_cpu":        {"reg": "ihub-irq-dev.message_5_dst_cpu",   "privatedata": "0x0"},
                "ihub_reset_irq_cpu":       {"reg": "ihub-irq-dev.message_4_dst_cpu",   "privatedata": "0x0"},
                "redun_notify_irq_dst_cpu": {"reg": "ihub-irq-dev.message_6_dst_cpu",   "privatedata": "0x1"}
            }
        },
    },
}
```

7 start_app utilities:

```text
#!/bin/sh

# devtmpfs does not get automounted for initramfs
var=`grep "chrt" /sbin/init`
sed -i '/'"$var"'/d' /sbin/init
var=`grep "chrt" /isam/supervisor/scripts/app_run`
sed -i '/'"$var"'/d' /isam/supervisor/scripts/app_run
var1=`grep "mount \\$entity_file \\$entity_mount_dir" /isam/scripts/swm_common_functions.sh`
var2=`echo ${var1/mount/mount -t squashfs}`
sed -i 's/'"$var1"'/'"$var2"'/g' /isam/scripts/swm_common_functions.sh

exec /sbin/init "$@"
```

<hr>

### # why async programming model?
async programming is a way to handle concurrent operations without using multiple threads,
which is particularly useful for i/o-bound tasks like network operations, file operations, etc.

$ 1 basic steps of async model:  
1.1 future creation: when async func is called, it will return a future (the task to be done).  
1.2 execution runtime: package tokio is used to manage the futures, which use an eventloop to check & select the
future to make progress, while only one future is executed on a thread at a time.  
1.3 state machine: each async func is decorated as a state machine, when future func encounter a operation (i/o) that blocks,
it yield the control back to runtime to execute other future, and in sometime it resume.

$ 2 features of async model:  
1.1 async code is typically single-threaded.  
1.2 while multiple async task can run concurrently (not in parallel) on the same thread.  
1.3 runtime switch between task when a task is blocking.  
1.4 await point out where the function yield control back to runtime.  
1.5 actual parallelism can be achieved by execute runtime on multiple threads.

<hr>

### # `Box<dyn Trait>`: safe syntax for rust polymorphism
`Box<dyn Trait>` is used to create obj of certain type on heap, which implemented Trait interfaces.
Box: a smart pointer assign to the allocated memory on heap, which allow dynamic mem & ownership management of data.
dyn Trait: the 'type' of obj to be allocated on heap.

unlike c/cpp, in golang or rust, a type is just collection of methods (interface or trait).
the `Box<dyn Trait>` allow dynamic type dispatch: different type implement the same trait can be allocated here,
hence, the exact type related trait method to call is determined at runtime rather than compile time.

<hr>

### # unwrap(): extract value from `Option<T>` & `Result<T, E>`
1 `Option<T>` is an enum that can either be Some(v) (indicating a value is present) or None (indicating no value).
using unwrap() on `Option<T>` will return value in Some(v) if exist, or program will panic if None.

```text
let some_value: Option<i32> = Some(10);
let value = some_value.unwrap();                            // value = 10

let none_value: Option<i32> = None;
let value = none_value.unwrap();                            // panic
```

2 `Result<T, E>` is an enum that can either be Ok(v) (indicating success and containing a value) or
Err(E) (indicating failure and containing an error).
using unwrap() on a `Result<T, E>` will return the value inside Ok if it exists, or program will panic if Err.

```text
let success: Result<i32, &str> = Ok(20);
let value = success.unwrap();                               // value is 20

let failure: Result<i32, &str> = Err("An error occurred");
let value = failure.unwrap();                               // panic
```
