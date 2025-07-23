I was wondering what makes it possible for language model (LLM) interfaces to load words gradually. Initially, I thought about using **WebSockets**, but then I realized that many implementations rely on **HTTP streaming** and **Server-Sent Events (SSE)**. This led me to dive deeper into SSE and its implementation, which is particularly well-suited for this purpose. In this post, I'll guide you through understanding SSE and building a simple example using the Golang Echo framework. We'll also model word generation intervals and create a complete implementation hosted on GitHub.

## Introduction to SSE

Server-Sent Events (SSE) is a **server push technology** enabling a server to push real-time updates to a web client over a single HTTP connection. Unlike WebSockets, SSE uses a one-way connection from the server to the client, which is ideal for scenarios where the client only needs to receive updates from the server.

Here’s a basic example to illustrate how SSE works using the Golang Echo framework. First, let's set up our server:

### Simple Example with Golang Echo Framework
```gist
<script src="https://gist.github.com/farid-moradi/64114ce6d8fa78ef4977f0431dff0084.js"></script>
```
This code sets up a simple SSE endpoint that sends a message to the client every second. You can see the updates in real-time in the browser’s network inspector.

For SSE, specific **HTTP headers** need to be set in the server's response to properly communicate with the client. Here are the key headers that should be included:

- **Content-Type:** This header must be set to `text/event-stream` to indicate that the response is an SSE stream.
- **Cache-Control:** This header should typically be set to `no-cache` to prevent the response from being cached by proxies or the browser.
- **Connection:** This header should be set to `keep-alive` to ensure the connection remains open for as long as possible.

In SSE, the server sends a specially formatted stream of text to the client. Each message in this stream is preceded by `"data: "` followed by the actual data content of the message. Here's how it works:

- **"data: "**: This prefix indicates that the following text is the payload of the SSE message. It's a key part of the SSE protocol that ensures the client can properly parse and handle the incoming data.
- **"\n\n"**: Each message is terminated by a double newline. This indicates the end of a single message. The client waits for this delimiter before processing the message.

In the example, the line `fmt.Fprintf(c.Response(), "data: %s\n\n", wordList[i])` sends a string formatted as `"data: %s\n\n"` where the payload is a word in `wordList`. The `c.Response().Flush()` call ensures that the message is immediately sent to the client without buffering.

To visualize how SSE messages are received by the client, you can open your browser’s developer tools and inspect the Network tab. You’ll see individual SSE messages being streamed in real-time. Here you can see that the entire response finishes in 2 s; the first word arrives after 1 s:

![Network Inspector](https://api.devfmd.xyz/uploads/a8fd9673-eac3-4fdb-b509-5413bc6715fa.png)

Here you can see the messages in the EventStream tab in the network inspect in the browser:

![Network Inspector](https://api.devfmd.xyz/uploads/547b692f-faab-46f5-8188-8c36148b8fe9.png)

## Mimicking Word Generation

To mimic the process of word generation, we can model the time interval between generating the next word in milliseconds as a random variable that follows an exponential distribution with a **rate parameter** of 0.07, and a **location shift** of 100 milliseconds as the minimum. This creates a realistic simulation of the gradual word generation seen in LLM interfaces. A rate parameter of 0.07 means the mean of the distribution is approximately 14.29 milliseconds, so the expected interval is 100 ms + 14.29 ms ≈ 114 ms.

Here's the Python code to plot the exponential distribution of the random variable we are examining:
```gist
<script src="https://gist.github.com/farid-moradi/b51c827fe99be6935b779150c38277c4.js"></script>
```

This generates the following plot:
![Exponential Distribution](https://api.devfmd.xyz/uploads/c05832ac-550f-41d0-ab80-e7fbe1527519.png)

You can see the random values generated from this distribution start at 100 ms, and as we move further to the right, the probability of taking longer for the next word to be generated decreases. This is similar to what we see in LLM interfaces where most words appear quickly, but occasionally it takes longer for the next word to load. It's worth noting that our assumption about the random variable following the exponential distribution is simplistic, given the independence premise of the exponential distribution, but it serves our needs in this example.

## Final Implementation and GitHub Hosting

To see the complete implementation and run it yourself, you can find the project hosted on GitHub: [GitHub Repository](https://github.com/farid-moradi/streaming-text-generation)

Below is a snippet that streams a string:

```gist
<script src="https://gist.github.com/farid-moradi/64de111992d05a37696e6f6c771644e2.js"></script>
```

We defined a new function named `ExpRand` to generate random numbers based on the exponential distribution we talked about. In the main function, we create a new Echo instance and set some general headers. Then we define a route named `stream` that accepts requests and uses the `Fields` method to split the `text` string into slices of substrings. Then we send the string word by word to the client. Finally, we start the Echo server to listen on port `8080`.

In conclusion, by using SSE and modeling word generation with an exponential distribution, we can effectively create a smooth and realistic text streaming experience. Feel free to explore and modify the code to suit your needs!
