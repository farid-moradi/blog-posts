
# Streaming Text Generation with SSE in Golang: Building an LLM Interface

I was wondering what makes it possible for language model (LLM) interfaces to load words gradually. Initially, I thought about using **WebSockets**, but then I realized that many implementations rely on **HTTP streaming** and **Server-Sent Events (SSE)**. This led me to dive deeper into SSE and its implementation, which is particularly well-suited for this purpose. In this post, I'll guide you through understanding SSE and building a simple example using the Golang Echo framework. We'll also model word generation intervals and create a complete implementation hosted on GitHub.

## Introduction to SSE

Server-Sent Events (SSE) is a **server push technology** enabling a server to push real-time updates to a web client over a single HTTP connection. Unlike WebSockets, SSE uses a one-way connection from the server to the client, which is ideal for scenarios where the client only needs to receive updates from the server.

Here’s a basic example to illustrate how SSE works using the Golang Echo framework. First, let's set up our server:

### Simple Example with Golang Echo Framework

```go
package main

import (
	"fmt"
	"net/http"
	"time"

	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"
)

func main() {
	// Create a new Echo instance
	e := echo.New()

	e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
		AllowOrigins: []string{"*"}, // Replace with your allowed origins
	}))

	// Define a route to handle the incremental text streaming
	e.GET("/stream", func(c echo.Context) error {
		c.Response().Header().Set("Content-Type", "text/event-stream")
		c.Response().Header().Set("Cache-Control", "no-cache")
		c.Response().Header().Set("Connection", "keep-alive")
		c.Response().WriteHeader(http.StatusOK)

		var wordList = []string{
			"LLM", "Interface",
		}

		// Send text incrementally every second
		for i := 0; i < len(wordList); i++ {
			fmt.Fprintf(c.Response(), "data: %s\n\n", wordList[i])
			c.Response().Flush()
			time.Sleep(time.Second)
		}

		return nil
	})

	// Start the Echo server
	e.Start(":8080")
}
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

To visualize how SSE messages are received by the client, you can open your browser’s developer tools and inspect the Network tab. You’ll see individual SSE messages being streamed in real-time. Here you can see that it takes 2 seconds for the client to receive the response from the server (one second for each of the 2 words):

![Network Inspector](https://api.devfmd.xyz/uploads/a8fd9673-eac3-4fdb-b509-5413bc6715fa.png)

Here you can see the messages in the EventStream tab in the network inspect in the browser:

![Network Inspector](https://api.devfmd.xyz/uploads/547b692f-faab-46f5-8188-8c36148b8fe9.png)

## Mimicking Word Generation

To mimic the process of word generation, we can model the time interval between generating the next word in milliseconds as a random variable that follows an exponential distribution with a **rate parameter** of 0.07, and a **location shift** of 100 milliseconds as the minimum. This creates a realistic simulation of the gradual word generation seen in LLM interfaces. A rate parameter of 0.07 means the mean of the distribution is approximately 14.29 milliseconds, adding to the minimum of 100 ms.

Here's the Python code to plot the exponential distribution of the random variable we are examining:

```python
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# Set the random seed for reproducibility
np.random.seed(42)

# Define the rate parameter (lambda) and the shift
lambda_param = 0.07  # Rate parameter for the exponential distribution
shift = 100  # Minimum interval time in milliseconds

# Generate random intervals using the exponential distribution and apply the shift
num_intervals = 10000  # Number of intervals to generate
intervals = np.random.exponential(scale=1/lambda_param, size=num_intervals) + shift

# Plot the histogram of the generated intervals
plt.figure(figsize=(10, 6))
sns.histplot(intervals, bins=30, kde=True, stat="density", color='skyblue', label='Histogram of Intervals')

# Plot the theoretical PDF of the shifted exponential distribution
x = np.linspace(shift, max(intervals), 10000)
pdf = lambda_param * np.exp(-lambda_param * (x - shift))
plt.plot(x, pdf, 'r-', lw=2, label='Theoretical Shifted Exponential PDF')

# Add labels and legend
plt.xlabel('Interval Time (ms)')
plt.ylabel('Density')
plt.title('Histogram and Theoretical PDF of Shifted Exponential Distribution')
plt.legend()
plt.grid(True)

# Show the plot
plt.show()
```

This generates the following plot:
![Exponential Distribution](https://api.devfmd.xyz/uploads/c05832ac-550f-41d0-ab80-e7fbe1527519.png)

You can see the random values generated from this distribution start at 100 ms, and as we move further to the right, the probability of taking longer for the next word to be generated decreases. This is similar to what we see in LLM interfaces where most words appear quickly, but occasionally it takes longer for the next word to load. It's worth noting that our assumption about the random variable following the exponential distribution is simplistic, given the independence premise of the exponential distribution, but it serves our needs in this example.

## Final Implementation and GitHub Hosting

To see the complete implementation and run it yourself, you can find the project hosted on GitHub: [GitHub Repository](https://github.com/yourusername/streaming-text-generation)

Below is a snippet that streams a string:

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"strings"
	"time"

	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"
)

const text = `
This repository contains code examples and an implementation for building a streaming text generation interface using Server-Sent Events (SSE) in Golang. SSE is a lightweight protocol for real-time communication over HTTP, ideal for scenarios like Large Language Model (LLM) interfaces.
`

// ExpRand generates a random value from an exponential distribution
// with the given rate parameter (lambda) and location shift (mu).
func ExpRand(lambda, mu float64) float64 {
	// Generate a random value from the standard exponential distribution
	u := rand.ExpFloat64()

	// Apply the rate parameter (lambda)
	x := u / lambda

	// Add the location shift (mu)
	return x + mu
}

func main() {
	// Create a new Echo instance
	e := echo.New()

	e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
		AllowOrigins: []string{"*"}, // Replace with your allowed origins
	}))

	// Define a route to handle the incremental text streaming
	e.GET("/stream", func(c echo.Context) error {
		c.Response().Header().Set("Content-Type", "text/event-stream")
		c.Response().Header().Set("Cache-Control", "no-cache")
		c.Response().Header().Set("Connection", "keep-alive")
		c.Response().WriteHeader(http.StatusOK)

		words := strings.Fields(text)
		for _, word := range words {
			fmt.Fprintf(c.Response(), "data: %s\n\n", word)
			c.Response().Flush()
			time.Sleep(time.Duration(ExpRand(0.07, 100)) * time.Millisecond)
		}

		return nil
	})

	// Start the Echo server
	e

.Start(":8080")
}
```


We defined a new function named `ExpRand` to generate random numbers based on the exponential distribution we talked about. In the main function, we create a new Echo instance and set some general headers. Then we define a route named `stream` that accepts requests and uses the `Fields` method to split the `text` string into slices of substrings. Then we send the string word by word to the client. Finally, we start the Echo server to listen on port `8080`.

In conclusion, by using SSE and modeling word generation with an exponential distribution, we can effectively create a smooth and realistic text streaming experience. Feel free to explore and modify the code to suit your needs!