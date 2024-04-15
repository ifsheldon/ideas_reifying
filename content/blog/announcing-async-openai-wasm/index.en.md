+++
title = "Announcing async-openai-wasm, and thoughts on wasmization and streams"
description = "Async Rust library for interacting with OpenAI's APIs on WASM and how I did it"
draft = false

weight = 4

[taxonomies]
tags = ["Rust", "WebAssembly", "OpenAI"]

[extra]
feature_image = "crab_and_robot_hand.png"
feature = true
+++

> 中文版请见[链接](@/blog/announcing-async-openai-wasm/index.md)

Today, I'm excited to announce the release of [`async-openai-wasm`](https://github.com/ifsheldon/async-openai-wasm)!

`async-openai-wasm` is a fork of [`async-openai`](https://github.com/64bit/async-openai/) and now has stable support for WebAssembly. With it, you can interact with OpenAI's APIs and use it in your WebAssembly projects.
It targets `wasm32-unknown-unknown`, so basically you can use it in any WebAssembly projects.
For example, you can now ship frontend-only apps that have AI superpowers without a backend server. You can also develop AI agents that run on edge functions like those on Cloudflare Workers.

> HELP WANTED:
>
> If you are interested in contributing, please check out the [GitHub repository](https://github.com/ifsheldon/async-openai-wasm).
>
> For now, the most wanted is to bring back `backoff` for exponential backing off requests, which is incompatible with `wasm32-unknown-unknown` due to the use of `tokio`/`async-std` functions.

Well, the above is basically all the announcement, but this is only *What*, let's talk about *Why* and *How*.

## Why `async-openai-wasm`

[`async-openai`](https://github.com/64bit/async-openai/) is an awesome crate that allows you to interact with OpenAI's APIs in async Rust. However, it doesn't have stable support for WebAssembly.
I've been maintaining an experimental branch of it, in which WebAssembly is supported behind a feature gate.
However, that means to use it in a WASM project, you need to download the crate by specifying the git repository and branch in your `Cargo.toml`.
That also prevents you to publish your project that depends on WASM feature of `async-openai` to [crates.io](https://crates.io).

## How to wasmize `async-openai`

When it comes to the combination of **async** Rust and WebAssembly, the first major problem is often async runtimes, like `tokio`, which usually have no or very limited support for WebAssembly.

In the case of `async-openai`, getting rid of `tokio` was ultimately reduced to one function `stream` that transforms an `EventSource` into a `Stream` of responses.

```rust
/// Request which responds with SSE.
pub(crate) async fn stream<O>(
    mut event_source: EventSource,
) -> Pin<Box<dyn Stream<Item = Result<O, OpenAIError>> + Send>>
where
    O: DeserializeOwned + std::marker::Send + 'static,
{
    let (tx, rx) = tokio::sync::mpsc::unbounded_channel();

    tokio::spawn(async move {
        while let Some(ev) = event_source.next().await {
            match ev {
                Err(e) => {
                    if let Err(_e) = tx.send(Err(OpenAIError::StreamError(e.to_string()))) {
                        // rx dropped
                        break;
                    }
                }
                Ok(event) => match event {
                    Event::Message(message) => {
                        if message.data == "[DONE]" {
                            break;
                        }

                        let response = match serde_json::from_str::<O>(&message.data) {
                            Err(e) => Err(map_deserialization_error(e, message.data.as_bytes())),
                            Ok(output) => Ok(output),
                        };

                        if let Err(_e) = tx.send(response) {
                            // rx dropped
                            break;
                        }
                    }
                    Event::Open => continue,
                },
            }
        }

        event_source.close();
    });

    Box::pin(tokio_stream::wrappers::UnboundedReceiverStream::new(rx))
}
```

OpenAI uses [Server-Sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events#event_stream_format) to stream responses. The streamed payloads conceptually flows like below:

![event_flow](event_flow.png)

OpenAI first sends an `OPEN` event to indicate the start of the stream, then sends a series of `MESSAGE` events, each of which contains a JSON string representing a response. Finally, it sends a `MESSAGE` event with a string `"[DONE]"` to indicate the end of the stream.

The consumer usually consumes the stream in a loop like below:

```rust
while let Some(chunk) = stream.next().await {
    match chunk {
        Ok(response) => {
            // do something with the response
        }
        Err(e) => log::error!("OpenAI Error: {:?}", e),
    }
}
```

The contract is that the stream continues with `Some(O)` (where `O` is deserialized from a string) and ends with a `None`.

The overall logic is simple, but the implementation is a bit involved. In the original implementation, `tokio` is used to spawn a task that listens to the SSE stream and sends responses to a channel. The channel receiver `rx` is then converted to a `Stream` that can be consumed by the caller.
Conceptually, there are two concurrent "threads" like below:

![async_openai_stream_implementation](asyn_openai_stream_implementation.png)

Since in essence, the function is just a transformation from `EventSource` to `Stream`, we can make a custom struct that implements `Stream`, which polls the `EventSource` and yields responses.

Here is what I made:

```rust
use futures::{stream::StreamExt, Stream};
use futures::stream::Filter;
use reqwest_eventsource::{Event, EventSource, RequestBuilderExt};
use std::future;
use std::marker::PhantomData;
use std::pin::Pin;
use std::task::{Context, Poll};
use pin_project_lite::pin_project;

pin_project! {
     pub struct OpenAIEventStream<O> {
        #[pin]
        stream: Filter<EventSource, future::Ready<bool>, fn(&Result<Event, reqwest_eventsource::Error>) -> future::Ready<bool>>,
        // to make the struct generic, which is needed for the Stream trait to customize the output type.
        _phantom_data: PhantomData<O> 
    }
}

impl<O> OpenAIEventStream<O> {
    pub(crate) fn new(event_source: EventSource) -> Self {
        Self {
            stream: event_source.filter(|result|
                // filter out the first event which is always Event::Open
                future::ready(!(result.is_ok()&&result.as_ref().unwrap().eq(&Event::Open)))
            ),
            _phantom_data: PhantomData
        }
    }
}

impl<O: DeserializeOwned + Send + 'static> Stream for OpenAIEventStream<O> {
    type Item = Result<O, OpenAIError>;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let this = self.project();
        let stream: Pin<&mut _> = this.stream;
        match stream.poll_next(cx) {
            Poll::Ready(response) => {
                match response {
                    None => Poll::Ready(None), // end of the stream
                    Some(result) => match result {
                        Ok(event) => match event {
                            Event::Open => unreachable!(), // it has been filtered out
                            Event::Message(message) => {
                                if message.data == "[DONE]" {
                                    Poll::Ready(None)  // end of the stream, defined by OpenAI
                                } else {
                                    // deserialize the data
                                    match serde_json::from_str::<O>(&message.data) {
                                        Err(e) => Poll::Ready(Some(Err(map_deserialization_error(e, &message.data.as_bytes())))),
                                        Ok(output) => Poll::Ready(Some(Ok(output))),
                                    }
                                }
                            }
                        }
                        Err(e) => Poll::Ready(Some(Err(OpenAIError::StreamError(e.to_string()))))
                    }
                }
            }
            Poll::Pending => Poll::Pending
        }
    }
}
```

> You can see there are many `pin`s, but I won't go into details here. For `pin`s and futures, here is a great [blog](https://blog.adamchalmers.com/pin-unpin/).

`OpenAIEventStream<O>` stores a `Filter` wrapping an `EventSource` instead. But for now let's just focus on the `poll_next` method. It's a bit more verbose, but it doesn't require `tokio`.

When we poll the wrapped event source, if it is ready and returns `None`, then the stream exhausted and we return `None` to signal the end of the stream. Similarly, according to OpenAI's API, if the data is `"[DONE]"`, we should also return `None`.
If we get a message that is not `"[DONE]"`, we try to deserialize it and return the deserialized value or any errors. These results should be wrapped in `Poll::Ready` since the event source is indeed ready and gives us a response.

The only branch that is slightly complicated to reason about is `Event::Open => ...`.
Given that the event source is ready and gives us some results, should we return `Poll::Pending` or `Poll::Ready`? If we return `Poll::Pending`, the stream will be scheduled to be poll again LATER, which is not what we want.
And I've tried this, it doesn't work as responses are delayed significantly. So, should we return `Poll::Ready`? Well then we need to return something, but what? `None`? But it's not the end of the stream. Then `Some` seems okay, but we don't have a valid message yet.

If we think out of the above frame, we should just filter out `Event::Open` as the stream consumer doesn't care about it. So, we use `StreamExt::filter` that is implemented by `EventSource` to filter out the `Event::Open` event.
That method returns a scary `Filter<EventSource, future::Ready<bool>, fn(&Result<Event, reqwest_eventsource::Error>) -> future::Ready<bool>>` which we fearlessly store in `OpenAIEventStream`.

Now with the new `OpenAIEventStream`, we don't need another spawned task to listen to the event source. We can just poll it directly and the execution is composed into the caller's task.

> For those who are curious, you can dig into `Stream` impl of `futures::stream::Filter` and see how it works.
> Simply put, in the `poll_next` method, it **eagerly** polls the inner stream in a `loop` and keeps looking for a result that satisfies the predicate until finding one or the stream exhausts.
>
> We could of course do that by ourselves if we don't want to store the `Filter` in `OpenAIEventStream`, but the code would be a lot verbose and difficult to understand.

The second challenge is related to file I/O, which is easier to solve. In the original implementation, `tokio` is also used to read and write files. But on `wasm32-unknown-unknown`, we can't use file I/O directly,
since the compiled binary may run in a browser or a serverless environment where file I/O is not allowed or not practical. So, we remove all file I/O related code and expose APIs that accept raw bytes in memory. I didn't do this.
Or more precisely, I did destructively. I just removed all file I/O related code. Thanks to the generous contributor in this [PR](https://github.com/64bit/async-openai/pull/154), we have in-memory APIs now.

## Closing

I hope this post gives you some insights into how to wasmize async Rust libraries. It's not that hard, but it requires some understanding of async programming and the Rust ecosystem.

If you think you have better solutions for the above problems, please definitely let me know!

## Metadata

Version: 1.0.0

Date: 2024.04.16

License: [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

