+++
title = "Slack Your REST Client with a couple of Serde Tricks"
description = "A couple of serde tricks allows a REST client to opt-in more flexibility while still by default complies to an OpenAPI spec"
draft = false

weight = 9

[taxonomies]
tags = ["Rust"]

[extra]
feature_image = "slack.png"
feature = true
+++

I am maintaining the [async-openai-wasm](https://crates.io/crates/async-openai-wasm) crate, which _was_ a client to access OpenAI services only. It was a bit rigid because request and response types are hardened in types,
but we need a bit more flexibility for it to support LLM services that claim to be OpenAI-API-compatible. 

More specifically, the client should be able to send extra data to a LLM service provider in addition to the parameters defined by OpenAI's OpenAPI specs.
After parsing spec-compliant part of the data in the response from the provider, it should also retain additional fields that may be arbitrary.

## Announcing `async-openai-wasm 0.27.3`

With the release `async-openai-wasm 0.27.3`, you can use it to access other OpenAI-API-compatible LLM services 🎉

You can access a recent blockbuster, DeepSeek R1, via the official Deepseek endpoint, OpenRouter or whichever provider of your choice. 
For example, OpenRouter [requires](https://openrouter.ai/docs/api-reference/parameters#include-reasoning) an additional field `include_reasoning` to retrieve `reasoning` content in the response of DeepSeek R1 model.

The code is just a few dozens of lines:
```rust
use async_openai_wasm::config::OpenAIConfig;
use async_openai_wasm::types::{
    ChatCompletionRequestUserMessageArgs, CreateChatCompletionRequestArgs,
};
use async_openai_wasm::Client;
use futures::StreamExt;
use serde_json::json;

const OPENROUTER_REASONING_KEY: &str = "reasoning";
const OPENROUTER_BASEURL: &str = "https://openrouter.ai/api/v1";

#[tokio::main]
async fn main() {
    let test_key = std::env::var("TEST_API_KEY").unwrap();
    let client = Client::with_config(
        OpenAIConfig::new()
            .with_api_base(OPENROUTER_BASEURL)
            .with_api_key(test_key),
    );
    let request = CreateChatCompletionRequestArgs::default()
        .messages(vec![ChatCompletionRequestUserMessageArgs::default()
            .content("Hello! Do you know the Rust programming language?")
            .build()
            .unwrap()
            .into()])
        .model("deepseek/deepseek-r1")
        // The extra params that OpenRouter requires to get reasoning content
        // See https://openrouter.ai/docs/api-reference/parameters#include-reasoning
        .extra_params(json!({
            "include_reasoning" : true
        }))
        .build()
        .unwrap();
    let mut result = client.chat().create_stream(request).await.unwrap();
    let mut reasoning = String::new();
    while let Some(result) = result.next().await {
        if let Ok(r) = result {
            // Get the reasoning field in the response
            let catch_all_return = r.choices[0].delta.return_catchall.as_ref();
            let reasoning_part = catch_all_return
                .and_then(|val| val.get(OPENROUTER_REASONING_KEY))
                .and_then(|r| r.as_str());
            if let Some(reasoning_part) = reasoning_part {
                reasoning.push_str(reasoning_part);
                println!("Reasoning Part: {reasoning_part}")
            }
        }
    }
    println!("Reasoning:\n{reasoning}");
}
```

Both of `include_reasoning` and `reasoning` fields are not in OpenAI's OpenAPI specs, so prior versions of `async-openai-wasm` had no way to add these to requests and parse them from responses.

This was fine until recently I kinda want to use my crate access DeepSeek R1.

## Give Your REST API Client a Little Slack

I need to extend my crate a bit, so it can be flexible and adaptable to upcoming models and services, maybe DeepSeek R2 :) 

```rust
#[derive(Clone, Serialize, Default, Debug, Builder, Deserialize, PartialEq)]
pub struct CreateChatCompletionRequest {
    // other parameters, 100 lines zipped ..... 

    #[serde(skip_serializing_if = "Option::is_none")]
    #[serde(flatten)]
    pub extra_params: Option<serde_json::Value>,
}

#[derive(Debug, Deserialize, Serialize, Clone, PartialEq)]
pub struct ChatCompletionResponseMessage {
    // other fields, 20 lines zipped ..... 

    /// Catching anything else that a provider wants to provide, for example, a `reasoning` field
    #[serde(flatten)]
    pub return_catchall: Option<serde_json::Value>,
}
```

I added `extra_params` to my request type and `return_catchall` to the response type. 

`#[serde(skip_serializing_if = "Option::is_none")]` ensures the request is compliant to the spec when `extra_params` is `None`. 

`#[serde(flatten)]` flattens the field. By "flatten", it means:
* In serialization, the fields in `extra_params` are leveled up. 
* In deserialization, `serde` will try to fit any left-out fields at the _same level_ into `return_catchall`

The below example demonstrates the behaviors more clearly:

```Rust
#[derive(Serialize, Deserialize)]
pub struct Msg<T: Serialize> {
    pub required_field: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    #[serde(flatten)]
    pub extra_data: Option<T>
}

#[derive(Serialize)]
pub struct RequestExtraData {
    pub a: i8,
    pub b: String
}

#[test]
fn test_container_serde() {
    // Serialization
    let request_data = RequestExtraData {
        a: 1,
        b: "test_data".to_string()
    };
    let request_container = Msg {
        required_field: "test_container".to_string(),
        extra_data: Some(request_data)
    };
    assert_eq!(
         serde_json::to_string(&request_container).unwrap(),
        r#"{"required_field":"test_container","a":1,"b":"test_data"}"#
    );
    
    // Deserialization
    // The below line is dangerous if the payload is maliciously large. Define a proper struct if you don't trust a server
    type ResponseExtraData = serde_json::Value; 
    let json_string = r#"
{
    "required_field": "test",
    "extra_int": 1,
    "extra_float": 1.0,
    "extra_string": "str"
}"#;
    let response_container: Msg<ResponseExtraData> = serde_json::from_str(json_string).unwrap();
    assert_eq!(
        serde_json::to_string(&response_container).unwrap(), 
        r#"{"required_field":"test","extra_float":1.0,"extra_int":1,"extra_string":"str"}"#
    );
}
```

In the serialization case, `request_container` is serialized into 
```json
{
   "required_field": "test_container",
   "a": 1,
   "b": "test_data"
}
```
instead of 
```json
{
   "required_field": "test_container",
   "extra_data":{
      "a": 1,
      "b": "test_data"
   }
}
```

In the deserialization case, the fields `extra_int`, `extra_float` and `extra_string` are extra fields at the same level as `extra_data`, so they are packed into a `ResponseExtraData`, which is `serde_json::Value` in our case. So, the values in `extra_data` are
```
"extra_int": 1,
"extra_float": 1.0,
"extra_string": "str"
```

Note that making a catch-all field as `serde_json::Value` in a response type is **dangerous**. If the payload is maliciously large, this may crash your server by using up its memory. Do this when you trust your service provider. 
Otherwise, you can still extend the message container type with your own types, just like `Msg<RequestExtraData>`.

> For your information: [serde documentation](https://serde.rs/attr-flatten.html) provides a few more examples of struct flattening.

## Conclude

To conclude, the combo of `Option`, `#[serde(skip_serializing_if = "Option::is_none")]` and `#[serde(flatten)]` allows a REST client to opt-in more flexibility while still by default complies to an OpenAPI spec. 

Have fun hacking!

## Metadata

Version: 0.0.1

Date: 2025.02.19

License: [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)
