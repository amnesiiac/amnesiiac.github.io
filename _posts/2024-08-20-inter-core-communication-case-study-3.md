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
rpmsg.rs:

```blurtext
use crate::inotify::REDUN_NOTIFY;
use async_trait::async_trait;
use pnet_datalink::{self, Channel, DataLinkReceiver, DataLinkSender, MacAddr, NetworkInterface};
use pnet_packet::ethernet::{EtherType, EthernetPacket, MutableEthernetPacket};
use pnet_packet::Packet;
use std::io;
use std::str;
use std::sync::mpsc::{Receiver, Sender};
use tokio::time::{sleep, Duration};

const ETHERNET_HEADER_LEN: usize = 14;

#[async_trait]
pub trait Services {
    async fn watcher(&mut self, sync_tx: Sender<String>);
    async fn handler(&mut self, sync_rx: Receiver<String>);
}

pub struct RawSocket {
    pub interface: NetworkInterface,
    pub receiver: Box<dyn DataLinkReceiver>,
    pub sender: Box<dyn DataLinkSender>,
}

impl RawSocket {
    pub fn new(name: &str) -> Self {
        let interface_names_match = |iface: &NetworkInterface| iface.name == name;
        let interfaces = pnet_datalink::interfaces();
        let interface = interfaces
            .into_iter()
            .filter(interface_names_match)
            .next()
            .unwrap();

        let (tx, rx) = match pnet_datalink::channel(&interface, Default::default()) {
            Ok(Channel::Ethernet(tx, rx)) => (tx, rx),
            Ok(_) => panic!("Unhandled channel type"),
            Err(e) => panic!(
                "An error occurred when creating the datalink channel: {}",
                e
            ),
        };

        RawSocket {
            interface: interface,
            receiver: rx,
            sender: tx,
        }
    }

    pub fn send(&mut self, buf: &str) -> Option<io::Result<()>> {
        let len = buf.len() + ETHERNET_HEADER_LEN;
        println!("[rpmsg] send raw packet to {:?}", self.interface.name);
        return self.sender.build_and_send(1, len, &mut |packet| {
            let mut packet = MutableEthernetPacket::new(packet).unwrap();
            packet.set_destination(MacAddr::broadcast());
            packet.set_source(self.interface.mac.unwrap());
            packet.set_ethertype(EtherType(0x8800));
            packet.set_payload(buf.as_bytes().to_vec().as_slice());
        });
    }
}

#[async_trait]
impl Services for RawSocket {
    async fn watcher(&mut self, sync_tx: Sender<String>) {
        let mut error_printed = false;
        loop {
            match self.receiver.next() {
                Ok(packet) => {
                    let packet = EthernetPacket::new(packet).unwrap();
                    let payload = packet.payload().clone();

                    let dst = str::from_utf8(payload).expect("Invalid utf-8");
                    sync_tx.send(dst.to_string()).unwrap();
                    if error_printed {
                        eprintln!("Recovered from error");
                        error_printed = false;
                    }
                }
                Err(e) => {
                    if !error_printed {
                        eprintln!("An error occurred while reading: {}", e);
                        error_printed = true;
                    }
                    continue;
                }
            };
            sleep(Duration::from_millis(100)).await;
        }
    }

    async fn handler(&mut self, sync_rx: Receiver<String>) {
        loop {
            if let std::result::Result::Ok(dst) = sync_rx.try_recv() {
                match dst.as_str() {
                    REDUN_NOTIFY => self.send(REDUN_NOTIFY),
                    _ => None,
                };
            }
            sleep(Duration::from_millis(100)).await;
        }
    }
}
```

inotify.rs:

```blurtext
use crate::rpmsg::Services;
use async_trait::async_trait;
use inotify::Inotify;
use std::fs;
use std::fs::File;
use std::fs::OpenOptions;
use std::io;
use std::io::Write;
use std::path::Path;
use std::sync::mpsc::{Receiver, Sender};
use tokio::time::{sleep, Duration};

pub const REDUN_NOTIFY: &str = "redun_notify_irq";
pub const IHUB_HWWD: &str = "ihub_hwwd_irq";
pub const IHUB_RESET: &str = "ihub_reset_irq";

pub struct IrqDevs {
    pub redun_notify_path: String,
    pub ihub_hwwd_path: String,
    pub ihub_reset_path: String,
}

impl IrqDevs {
    pub fn new(path: &str) -> Self {
        let data = fs::read_to_string(path).expect("Unable to read file");
        let json: serde_json::Value =
            serde_json::from_str(&data).expect("JSON does not have correct format.");

        let ihub_irq_dev = json
            .get("file")
            .unwrap()
            .get("ihub-irq-dev")
            .unwrap()
            .get("path")
            .unwrap()
            .to_string()
            .replace("\"", "");
        let ihub_irq_dev_path = Path::new(ihub_irq_dev.as_str());
        fs::create_dir_all(ihub_irq_dev_path.join("")).unwrap();

        let ihub_config_dev_regs = json
            .get("misc")
            .unwrap()
            .get("ihub_config_dev")
            .unwrap()
            .get("regs")
            .unwrap();

        let mut redun_notify_trig = "message_6_trig".to_string();
        match ihub_config_dev_regs.get("redun_notify_trig") {
            Some(reg) => {
                let reg_value = reg.get("reg").unwrap().to_string();
                let split: Vec<&str> = reg_value.split(".").collect();
                redun_notify_trig = split[1].to_string().replace("\"", "");
            }
            None => println!("redun_notify_trig use default path"),
        }
        let redun_notify_trig_path = ihub_irq_dev_path.join(redun_notify_trig);
        File::create(redun_notify_trig_path.clone()).expect("redun_notify_trig file create failed");

        let mut ihub_hwwd_irq = "message_5_state".to_string();
        match ihub_config_dev_regs.get("ihub_hwwd_irq") {
            Some(reg) => {
                let reg_value = reg.get("reg").unwrap().to_string();
                let split: Vec<&str> = reg_value.split(".").collect();
                ihub_hwwd_irq = split[1].to_string().replace("\"", "");
            }
            None => println!("ihub_hwwd_irq use default path"),
        }
        let ihub_hwwd_irq_path = ihub_irq_dev_path.join(ihub_hwwd_irq);
        File::create(ihub_hwwd_irq_path.clone()).expect("ihub_hwwd_irq file create failed");

        let mut ihub_reset_irq = "message_4_state".to_string();
        match ihub_config_dev_regs.get("ihub_reset_irq") {
            Some(reg) => {
                let reg_value = reg.get("reg").unwrap().to_string();
                let split: Vec<&str> = reg_value.split(".").collect();
                ihub_reset_irq = split[1].to_string().replace("\"", "");
            }
            None => println!("ihub_reset_irq use default path"),
        }
        let ihub_reset_irq_path = ihub_irq_dev_path.join(ihub_reset_irq);
        File::create(ihub_reset_irq_path.clone()).expect("ihub_reset_irq file create failed");

        IrqDevs {
            redun_notify_path: redun_notify_trig_path.as_path().display().to_string(),
            ihub_hwwd_path: ihub_hwwd_irq_path.as_path().display().to_string(),
            ihub_reset_path: ihub_reset_irq_path.as_path().display().to_string(),
        }
    }

    fn write(&mut self, path: String, buf: &str) -> Option<io::Result<()>> {
        println!("[inotify] write file: {:?}", path);
        let mut f = OpenOptions::new()
            .append(false)
            .create(true)
            .write(true)
            .open(path)
            .expect("Unable to open file");
        f.write_all(buf.as_bytes()).expect("Unable to write data");

        Some(Ok(()))
    }
}

#[async_trait]
impl Services for IrqDevs {
    async fn watcher(&mut self, sync_tx: Sender<String>) {
        let mut inotify = Inotify::init().expect("Failed to initialize inotify");
        let redun_notify_descriptor = inotify
            .add_watch(
                self.redun_notify_path.clone(),
                inotify::WatchMask::CLOSE_WRITE,
            )
            .expect("Failed to add inotify watch for notify file");

        let mut buffer = [0; 32];
        loop {
            let events = inotify
                .read_events_blocking(&mut buffer)
                .expect("Failed to read inotify events");
            for event in events {
                if event.wd == redun_notify_descriptor {
                    println!("[inotify] redun_notify file modified: {:?}", event.name);
                    sync_tx.send(REDUN_NOTIFY.to_string()).unwrap();
                }
            }
            sleep(Duration::from_millis(100)).await;
        }
    }

    async fn handler(&mut self, sync_rx: Receiver<String>) {
        loop {
            if let std::result::Result::Ok(dst) = sync_rx.try_recv() {
                match dst.as_str() {
                    IHUB_HWWD => self.write(self.ihub_hwwd_path.clone(), "1"),
                    IHUB_RESET => self.write(self.ihub_reset_path.clone(), "1"),
                    _ => None,
                };
            }
            sleep(Duration::from_millis(100)).await;
        }
    }
}
```

main.rs:

```blurtext
use rpmsg::Services;                          // ?
use std::env;                                 // for access env var & cmdline arg
use std::sync::mpsc;                          // for multi-producer-single-consumer msg channel between threads

mod inotify;                                  // introduce inotify.rs content as mod
mod rpmsg;                                    // introduce rpmsg.rs content as mod

// micro tokio: mark main as async func, startup tokio runtime, allow await in it, allow async spawn task inside
#[tokio::main]
async fn main() {
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

cargo.toml:

```blurtext
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

usages:

```blurtext
$ UDRV_JSON_PATH=/isam/user/plat_srv/udrv.json start_app --bg /isam/user/host/apps/inco_proxy rpmsg4000 > /tmp/messages
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

### # code walk through inco-proxy app (cpp)
rpmsg.h:

```text
#include <iostream>
#include <string>
#include <thread>
#include <chrono>
#include <memory>
#include <vector>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <functional>
#include <optional>
#include <stdexcept>
#include <cstring>
#include <pnet_datalink/pnet_datalink.h>
#include <pnet_packet/pnet_packet.h>

const size_t ETHERNET_HEADER_LEN = 14;

class Services {
public:
    virtual void watcher(std::shared_ptr<std::queue<std::string>> sync_tx) = 0;
    virtual void handler(std::shared_ptr<std::queue<std::string>> sync_rx) = 0;
};

class RawSocket : public Services {
public:
    pnet_datalink::NetworkInterface interface;
    std::unique_ptr<pnet_datalink::DataLinkReceiver> receiver;
    std::unique_ptr<pnet_datalink::DataLinkSender> sender;

    RawSocket(const std::string& name){
        auto interfaces = pnet_datalink::interfaces();
        auto it = std::find_if(interfaces.begin(), interfaces.end(), [&](const pnet_datalink::NetworkInterface& iface){
            return iface.name == name;
        });
        if(it == interfaces.end()){
            throw std::runtime_error("Network interface not found");
        }
        interface = *it;
        auto channel = pnet_datalink::channel(interface, pnet_datalink::Default::default());
        if(std::holds_alternative<pnet_datalink::Channel>(channel)){
            auto [tx, rx] = std::get<pnet_datalink::Channel>(channel);
            sender = std::move(tx);
            receiver = std::move(rx);
        }
        else{
            throw std::runtime_error("Unhandled channel type");
        }
    }

    std::optional<std::exception_ptr> send(const std::string& buf){
        size_t len = buf.size() + ETHERNET_HEADER_LEN;
        std::cout << "[rpmsg] send raw packet to " << interface.name << std::endl;

        return sender->build_and_send(1, len, [&](uint8_t* packet){
            auto mutable_packet = pnet_packet::MutableEthernetPacket::new(packet);
            mutable_packet.set_destination(pnet_datalink::MacAddr::broadcast());
            mutable_packet.set_source(interface.mac.value());
            mutable_packet.set_ethertype(pnet_packet::EtherType(0x8800));
            mutable_packet.set_payload(reinterpret_cast<const uint8_t*>(buf.data()), buf.size());
        });
    }

    void watcher(std::shared_ptr<std::queue<std::string>> sync_tx) override {
        bool error_printed = false;
        while(true){
            auto packet = receiver->next();
            if(packet){
                auto ethernet_packet = pnet_packet::EthernetPacket::new(*packet);
                auto payload = ethernet_packet.payload();

                std::string dst(reinterpret_cast<const char*>(payload.data()), payload.size());
                sync_tx->push(dst);
                if(error_printed){
                    std::cerr << "Recovered from error" << std::endl;
                    error_printed = false;
                }
            }
            else{
                if(!error_printed){
                    std::cerr << "An error occurred while reading" << std::endl;
                    error_printed = true;
                }
                continue;
            }
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }

    void handler(std::shared_ptr<std::queue<std::string>> sync_rx) override {
        while(true){
            if(!sync_rx->empty()){
                std::string dst = sync_rx->front();
                sync_rx->pop();
                if(dst == "REDUN_NOTIFY"){
                    send("REDUN_NOTIFY");
                }
            }
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }
};
```

inotify.h:

```text
#include <iostream>
#include <fstream>
#include <string>
#include <thread>
#include <chrono>
#include <filesystem>
#include <json/json.h>
#include <condition_variable>
#include <mutex>
#include <queue>
#include <unordered_map>

class Services {
public:
    virtual void watcher(std::condition_variable& cv, std::queue<std::string>& sync_queue) = 0;
    virtual void handler(std::queue<std::string>& sync_queue) = 0;
};

const std::string REDUN_NOTIFY = "redun_notify_irq";
const std::string IHUB_HWWD = "ihub_hwwd_irq";
const std::string IHUB_RESET = "ihub_reset_irq";

class IrqDevs : public Services {
public:
    IrqDevs(const std::string& path){
        std::ifstream file(path);
        if(!file.is_open()){
            throw std::runtime_error("Unable to read file");
        }
        Json::Value json;
        file >> json;

        std::string ihub_irq_dev = json["file"]["ihub-irq-dev"]["path"].asString();
        std::filesystem::create_directory(ihub_irq_dev);
        auto ihub_config_dev_regs = json["misc"]["ihub_config_dev"]["regs"];

        std::string redun_notify_trig = "message_6_trig";
        if(ihub_config_dev_regs.isMember("redun_notify_trig")){
            std::string reg_value = ihub_config_dev_regs["redun_notify_trig"]["reg"].asString();
            redun_notify_trig = reg_value.substr(reg_value.find(".") + 1);
        }
        redun_notify_path = ihub_irq_dev + "/" + redun_notify_trig;
        std::ofstream(redun_notify_path).close();

        std::string ihub_hwwd_irq = "message_5_state";
        if(ihub_config_dev_regs.isMember("ihub_hwwd_irq")){
            std::string reg_value = ihub_config_dev_regs["ihub_hwwd_irq"]["reg"].asString();
            ihub_hwwd_irq = reg_value.substr(reg_value.find(".") + 1);
        }
        ihub_hwwd_path = ihub_irq_dev + "/" + ihub_hwwd_irq;
        std::ofstream(ihub_hwwd_path).close();

        std::string ihub_reset_irq = "message_4_state";
        if(ihub_config_dev_regs.isMember("ihub_reset_irq")){
            std::string reg_value = ihub_config_dev_regs["ihub_reset_irq"]["reg"].asString();
            ihub_reset_irq = reg_value.substr(reg_value.find(".") + 1);
        }
        ihub_reset_path = ihub_irq_dev + "/" + ihub_reset_irq;
        std::ofstream(ihub_reset_path).close();
    }

    void write(const std::string& path, const std::string& buf){
        std::cout << "[inotify] write file: " << path << std::endl;
        std::ofstream f(path);
        if(!f.is_open()){
            throw std::runtime_error("Unable to open file");
        }
        f << buf;
        f.close();
    }

    void watcher(std::condition_variable& cv, std::queue<std::string>& sync_queue) override {
        // Inotify implementation not provided in std cpp
        // 1 use fs polling or equivalent
        // 2 use custom inotify: ref: https://github.com/aabolfazl/inotify/blob/master/main.cpp
    }

    void handler(std::queue<std::string>& sync_queue) override {
        while(true){
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            std::unique_lock<std::mutex> lock(mutex);
            if(!sync_queue.empty()){
                std::string dst = sync_queue.front();
                sync_queue.pop();
                if(dst == IHUB_HWWD){
                    write(ihub_hwwd_path, "1");
                }
                else if(dst == IHUB_RESET){
                    write(ihub_reset_path, "1");
                }
            }
        }
    }

private:
    std::string redun_notify_path;
    std::string ihub_hwwd_path;
    std::string ihub_reset_path;
    std::mutex mutex;
};
```
