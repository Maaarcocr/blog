---
title: A story about async Rust and using !Send types
date: 2021-05-04T08:37:09.409Z
description: As I delved into integrating async Rust and wasmtime, I had
  to   figure out how to use !Send types while using a tokio runtime. In this
  blog   post I will summarise the issues I faced and my solution!
---
I was recently working on a hackathon project where I was trying to use wasmtime together with
tokio and tonic (gRPC rust library). I had a stream of messages coming in and I wanted to have
one single wasm module instance for the entire lifetime of the stream. Something like: 

```rust
async fn process_messages(
    &self,
    reqs: RequestStream<Message>,
) -> Result<tonic::Response<Self::ProcessMessagesStream>, Status> {
    let mut request_stream = Box::pin(reqs.into_inner());

    let response_stream = try_stream! {
        let instance = load_wasm_instance("some_wasm_file.wasm")
            .map_err(|e| Status::internal(e.to_string()))?;

        while let Some(value) = request_stream.next().await {
            let result = run_wasm_for_message(&value?, &instance)
                .map_err(|e| Status::internal(e.to_string()))?;
            yield result;
        }
    };

    Ok(tonic::Response::new(
        Box::pin(response_stream) as Self::ProcessMessagesStream
    ))
}
```

So I wanted to load a wasm file into a module instance for each stream of requests and always use that
same instance for each request that was coming in on the stream. This would allow the wasm module to keep its own state,
which was something I wanted, instead of creating a new instance everytime a new message arrived. 

The issue is that the code above does not compile. The error is: 

```
error: future cannot be sent between threads safely
   --> src\main.rs:150:13
    |
150 |             Box::pin(response_stream) as Self::ProcessMessagesStream
    |             ^^^^^^^^^^^^^^^^^^^^^^^^^ future created by async block is not `Send`
    |
    = help: within `AsyncStream<std::result::Result<MessageResponse, Status>, impl Future>`, the trait `Send` is not implemented for `Rc<wasmtime::store::StoreInner>`
note: future is not `Send` as this value is used across an await
   --> src\main.rs:138:31
    |
138 |           let response_stream = try_stream! {
    |  _______________________________^
139 | |             let instance = load_wasm_instance("some_wasm_file.wasm")
140 | |                 .map_err(|e| Status::internal(e.to_string()))?;
141 | |
...   |
146 | |             }
147 | |         };
    | |_________^ first, await occurs here, with `instance` maybe used later...
note: `instance` is later dropped here
   --> src\main.rs:147:9
    |
139 |             let instance = load_wasm_instance("some_wasm_file.wasm")
    |                 -------- has type `wasmtime::Instance` which is not `Send`
...
147 |         };
    |
```

The issue is that `wasmtime::Instance` does not implement the `Send` trait, which means that `wasmtime::Instance` cannot
be sent from one thread to another safely.

## Enter wasmtime::Instance

`wasmtime::Instance` is what you get after you load a wasm file with the wasmtime crate. It allows you to get some
metadata about the wasm module you have loaded, like what it exports and imports, but it also allows you
to call the exported functions from your rust code. It uses `Rc` quite extensively and I don't think `Instance` will ever be `Send`, more about it [here](https://github.com/bytecodealliance/wasmtime/issues/793).

I wanted to try to get this to work anyway, so I started googling how to possibly use something that is not `Send` together with tokio, tower and tonic.
I did end up reading that if I ensure that the Future is not sent across threads I would be able to still use tokio and async
stuff but also keep my single instance per stream idea feasible.

I just did not know how to exactly accomplish this and why I needed to do so.

## Why do we need `Send` in the first place?

One thing that was really bugging me is why did I even need everything to be `Send` in the first place. I thought that given
that the instance in created in its own tokio task (this is done by the async_stream macro), I would not need such a constraint! The
compiler was telling me that I was wrong, and I do trust the compiler so I tried to better understand what was going on.

```rust
// instance is created here and I do not see any
// threads being spawned where I would send the instance to
// so why does instance need to be Send????
let instance = load_wasm_instance("some_wasm_file.wasm")
    .map_err(|e| Status::internal(e.to_string()))?; 

while let Some(value) = request_stream.next().await {
    let result = run_wasm_for_message(&value?, &instance)
        .map_err(|e| Status::internal(e.to_string()))?;
    yield result;
}
```

The reason I needed everything to be `Send` is that tokio is a pretty clever runtime and it is multi-threaded. What happens behind the scenes is that
any `Future` which is currently blocked could then be awakened on a different thread than the one where it was being run before the block. So
everything need to be `Send` as tokio may send the `Future` and everything related to it (including the local variables that make up its state)
across different threads.

```rust
let instance = load_wasm_instance("some_wasm_file.wasm")
    .map_err(|e| Status::internal(e.to_string()))?; 

// when we call await we "block" the Future and when this "wakes up" again
// it may do so on a different thread. note that instance is part of this,
// as we use it inside the loop
while let Some(value) = request_stream.next().await {
    let result = run_wasm_for_message(&value?, &instance)
        .map_err(|e| Status::internal(e.to_string()))?;
    yield result;
}
```

## Actors and LocalSets

Now that I understood the root of the problem I wanted to come up with a nice solution to fix this. I was aware of the concept
of actors and my idea was to spawn an actor on a single thread and communicate with it through channels. The actor would own the
`wasmtime::Instance` and I would be able to run the function it exported by communicating with the actor itself.

As I said I was aware of the concept of actors but I did not know much about how to implement one using tokio, so I looked it up
on google and found a great blog post on this [here](https://ryhl.io/blog/actors-with-tokio/) (this is a great resource, you should read it!).

I followed it and I started defining my actor messages and the actor struct itself:

```rust
struct RunFunctionMessage {
    message: Message,
    respond_to: oneshot::Sender<Result<MessageResponse>>,
}

enum WasmInstanceActorMessage {
    Run(RunFunctionMessage),
}

struct WasmInstanceActor {
    receiver: mpsc::Receiver<WasmInstanceActorMessage>,
    instance: wasmtime::Instance,
}
```

I then implemented some methods on my actor: 

```rust
impl WasmInstanceActor {
    pub fn new(
        wasm_file: String,
        receiver: mpsc::Receiver<WasmInstanceActorMessage>,
    ) -> Result<Self, Error> {
        let instance = utils::load_wasm_instance(&wasm_file)?;
        Ok(Self { instance, receiver })
    }

    pub fn handle_message(&mut self, msg: WasmInstanceActorMessage) {
        match msg {
            WasmInstanceActorMessage::Run(run_function_message) => {
                // use the instance to call a function specificied by the message and get a result
                let result =
                    utils::run_wasm_for_message(&run_function_message.message, &self.instance);

                // send back the result using the oneshot channel provided in the message
                let _ = run_function_message.respond_to.send(result);
            }
        }
    }
}
```

and finally created a function that would run my actor, by listening to new messages from the
receiver and handling the with our `handle_message` method.

```rust
async fn run_wasm_instance_actor(mut actor: WasmInstanceActor) {
    while let Some(msg) = actor.receiver.recv().await {
        actor.handle_message(msg);
    }
}
```

### But how do I make all of this single-threaded for sure?

In the post that I linked to about creating a simple actor using tokio, the author did not focus
on the issue of creating an actor that is always run on a single thread (as it is a very
niche use case anyway).

The actor tutorial proceed to create an actor handler, which has the following responsabilities: 

- it starts a task where the actor runs
- it holds a reference to the channel with which to talk to the actor
- it exposes utility methods so that consumers do not need to send messages to the actor itself, but they can use the handler methods to do so

The thing that I had to come up with was how to start the task that runs my actor in such a way that the task only ever runs on a single thread.
The code I had from the tutorial was the following (this does not do what I want): 

```rust
#[derive(Clone)]
pub struct MyActorHandle {
    sender: mpsc::Sender<ActorMessage>,
}

impl MyActorHandle {
    pub fn new() -> Self {
        let (sender, receiver) = mpsc::channel(8);
        let actor = WasmInstanceActor::new(receiver);
        tokio::spawn(run_wasm_instance_actor(actor));

        Self { sender }
    }

    // some utility methods to talk to the actor...
}
```

The main issue here is that `tokio::spawn` just creates a new task and it schedules it on the default runtime. This is not what we want, as
it requires stuff to be `Send`. After some googling, I ended up reading the documentation for `LocalSet` (which you can find [here](https://docs.rs/tokio/1.4.0/tokio/task/struct.LocalSet.html)) which allows a set of tasks to be executed on the same thread. BINGO!

Following the documentation there I found a way of creating a local set while also being inside a tokio task (as I am, given that I'm handling gRPC requests using tonic). So I slightly changed my actor handler constructor such that: 

- it spawns a new OS thread
- on this new OS thread it creates a new tokio runtime that always run on the current thread
- created a local set and spawned a task in it which runs my actor
- profit (?)

The code to do this is: 

```rust
impl MyActorHandle {
    pub fn new(wasm_file: String) -> Self {
        let (sender, receiver) = mpsc::channel(8);
        std::thread::spawn(move || {
            let actor = WasmInstanceActor::new(wasm_file, receiver).unwrap();
            let rt = tokio::runtime::Builder::new_current_thread()
                .enable_all()
                .build()
                .unwrap();
            let local = tokio::task::LocalSet::new();
            local.spawn_local(run_wasm_instance_actor(actor));
            rt.block_on(local);
        });

        Self { sender }
    }

    pub async fn run_wasm_for_message(
        &self,
        message: Message,
    ) -> Result<MessageResponse> {
        let (send, recv) = oneshot::channel();
        let msg = WasmInstanceActorMessage::Run(RunFunctionMessage {
            message,
            respond_to: send,
        });

        let _ = self.sender.send(msg).await;
        recv.await.expect("Actor task has been killed")
    }
}
```

The original code, which now uses the actor handler looks like: 
```rust
async fn process_messages(
    &self,
    reqs: RequestStream<Message>,
) -> Result<tonic::Response<Self::ProcessMessagesStream>, Status> {
    let mut request_stream = Box::pin(reqs.into_inner());
    let instance_actor = MyActorHandle::new("some_wasm_file.wasm".to_string());
    let response_stream = try_stream! {
        while let Some(value) = request_stream.next().await {
            let result = instance_actor.run_wasm_for_message(value?)
                .await
                .map_err(|e| Status::internal(e.to_string()))?;
            yield result;
        }
    };

    Ok(tonic::Response::new(
        Box::pin(response_stream) as Self::ProcessMessagesStream
    ))
}
```

## Conclusion

We have learnt why we need types to be `Send` when using tokio, which is mostly due to the fact that, whenever you await something, you may find yourself in a different thread when you get woken up. Then we have learnt how to deal with types that are not `Send`
when using tokio (and other libraries that build on top of it). I explained a possible solution using an actor which runs on a
single thread using `LocalSet` and by spawning a new OS thread.