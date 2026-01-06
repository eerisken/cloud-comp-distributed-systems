# **Building a Bare-Metal Web Server with HermitOS**

This guide explores how to build and run a "Hello World" web server as a **unikernel** using **HermitOS**. In this model, your application and the operating system kernel are compiled into a single bootable image, eliminating the overhead of a traditional OS like Linux.

##  **Prerequisites**

To follow this guide, you will need a Linux environment (Ubuntu/Debian is recommended) and the Rust toolchain installed.

### **Install Rust Nightly and Hermit Target**

HermitOS relies on unstable features, so the nightly toolchain is required.

```Bash

# Install nightly toolchain  
rustup toolchain install nightly

# Add the HermitOS target  
rustup target add x86_64-unknown-hermit --toolchain nightly

# Install uhyve (The HermitOS Hypervisor)  
cargo install uhyve  
```

##  **The Application Code**

Create a new Rust project:

```Bash

cargo new hermit-web-server  
cd hermit-web-server  
```

Replace src/main.rs with this simple TCP server. HermitOS provides a compatibility layer for the Rust standard library, so your code looks like a standard networking app:

```Rust

use std::io::{Read, Write};  
use std::net::{TcpListener, TcpStream};

fn handle_client(mut stream: TcpStream) {  
    let mut buffer = [0; 1024];  
    let _ = stream.read(&mut buffer).unwrap();

    let response = "HTTP/1.1 200 OKrnContent-Type: text/plainrnrnHello World from HermitOS!";  
    stream.write_all(response.as_bytes()).unwrap();  
    stream.flush().unwrap();  
}

fn main() {  
    // We bind to 0.0.0.0 so it's accessible via the TAP interface  
    let listener = TcpListener::bind("0.0.0.0:8080").expect("Could not bind to port");  
    println!("Unikernel server listening on port 8080...");

    for stream in listener.incoming() {  
        match stream {  
            Ok(stream) => handle_client(stream),  
            Err(e) => println!("Connection error: {}", e),  
        }  
    }  
}  
```

##  **Networking: Setting up the TAP Interface**

Since the unikernel runs in a virtualized environment (via uhyve), it needs a way to communicate with your host machine. We use a **TAP interface**, which acts as a virtual Ethernet bridge.  
Run these commands on your host Linux machine:

```Bash

# Create a TAP interface named 'tap0'  
sudo ip tuntap add mode tap name tap0

# Assign an IP address to your host on this interface  
sudo ip addr add 10.0.5.1/24 dev tap0

# Bring the interface up  
sudo ip link set dev tap0 up

# (Optional) Enable masquerading if you want the unikernel to access the internet  
sudo iptables -t nat -A POSTROUTING -s 10.0.5.0/24 -j MASQUERADE  
```

##  **Building and Running**

### **Build the Unikernel**

Compile the binary for the Hermit architecture:

```Bash

cargo +nightly build --target x86_64-unknown-hermit --release  
```

### **Run with Uhyve**

When running uhyve, we must specify the network configuration so it knows to use the tap0 interface and what IP address the unikernel should assume.

```Bash

# HERMIT_IP: The IP of the unikernel  
# HERMIT_GATEWAY: The IP of your host (the tap0 interface)  
HERMIT_IP=10.0.5.2 HERMIT_GATEWAY=10.0.5.1 uhyve --netif tap0 target/x86_64-unknown-hermit/release/hermit-web-server
```
## **Verification**

Open a new terminal on your host machine and try to reach the unikernel:

```Bash
curl http://10.0.5.2:8080  
```  
Expected Output:  
Hello World from HermitOS!

##  **Why use a Unikernel?**

Traditional virtualization involves a heavy stack: Hardware -> Hypervisor -> Guest OS -> Application. Unikernels simplify this significantly.

* **Security:** Smaller attack surface. No shell, no SSH, and no unnecessary drivers.  
* **Performance:** Faster boot times (milliseconds) and lower memory footprint.  
* **Efficiency:** The binary only includes the "slices" of the OS it actually uses.

