# mio-echo
Another echo server with mio

```rust
extern crate mio;

use mio::net::{TcpListener, TcpStream};
use mio::unix::UnixReady;
use mio::*;
use std::collections::HashMap;
use std::collections::VecDeque;
use std::io::{Read, Write};
use std::sync::{Arc, Mutex};

use std::thread;
use std::time::Duration;

use std::error::Error;

type Connection = (TcpStream, VecDeque<Vec<u8>>, bool);

fn server() -> Result<(), Box<Error>> {
    let server_token = Token(0);
    let mut num_clients = 1;

    let addr = "127.0.0.1:13265".parse()?;

    // Setup the server socket

    let server = TcpListener::bind(&addr)?;

    // Create a list of clients to drop

    let mut droppable = Vec::new();

    // Create a poll instance

    let poll = Poll::new()?;

    // Start listening for incoming connections

    poll.register(&server, server_token, Ready::readable(), PollOpt::edge())?;

    let mut events = Events::with_capacity(1024);

    // Totally unneccessary to wrap this in an Arc'd mutex but I'm learning :)

    let clients: Arc<Mutex<HashMap<Token, Connection>>> = Arc::new(Mutex::new(HashMap::new()));

    let thread_copy = clients.clone();

    thread::spawn(move || loop {
        {
            let data = thread_copy.lock().unwrap();
            println!("There are {} clients connected", data.len());
        }
        thread::sleep(Duration::from_millis(1000));
    });

    println!("Starting up the echo server!");

    loop {
        poll.poll(&mut events, None)?;

        let mut client_map = clients.lock().unwrap();

        for event in &events {
            let token = event.token();

            if token == server_token {
                match server.accept() {
                    Ok((tcp_stream, socket_addr)) => {
                        println!("DEBUG: Got a new connection from {}", socket_addr);
                        let token = Token(num_clients);

                        poll.register(
                            &tcp_stream,
                            token,
                            Ready::readable() | Ready::writable() | UnixReady::hup(),
                            PollOpt::edge(),
                        )?;

                        client_map.insert(token, (tcp_stream, VecDeque::new(), false));
                        num_clients += 1;
                    }
                    Err(err) => println!("DEBUG: Err to accept: {}", err),
                }
            } else {
                match client_map.get_mut(&event.token()) {
                    Some((tcp_stream, queue, writable)) => {
                        println!(
                            "DEBUG: {:?} Got an event from client: {:?}",
                            event.token(),
                            event.readiness()
                        );

                        if event.readiness().contains(Ready::readable()) {
                            // Read as much as we can from the socket

                            let mut buff = [0; 1024];

                            match tcp_stream.read(&mut buff) {
                                Ok(n) if n > 0 => {
                                    println!("DEBUG: {:?} Read {} bytes", event.token(), n);
                                    queue.push_back(buff[0..n].to_vec());
                                }
                                _ => {}
                            }
                        }

                        if event.readiness().contains(Ready::writable()) || *writable {
                            *writable = true;

                            for queued in queue.drain(..) {
                                match tcp_stream.write(&queued) {
                                    Ok(n) if n > 0 => {
                                        println!("DEBUG: {:?} Wrote {} bytes", event.token(), n);
                                        if n < queued.len() {
                                            *writable = false;
                                        }
                                    }
                                    _ => {
                                        *writable = false;
                                        break;
                                    }
                                }
                            }
                        }

                        if event.readiness().contains(UnixReady::hup()) {
                            println!("DEBUG: {:?} Dropping client", token);
                            droppable.push(event.token());
                        }
                    }
                    None => println!("DEBUG: {:?} No client found", event.token()),
                }
            }
        }

        // Drop any droppable clients

        for token in droppable.drain(..) {
            client_map.remove_entry(&token);
        }
    }
}

fn main() {
    server().expect("Server error, shutting down...");
}
```
