---
title: "Testing Tips: Dependency Injection"
date: 2020-05-26T18:09:09+01:00
draft: true
summary: A strategy I used to improve the testability of a CLI application.
---

I've been writing [a card game][Shed] and I planned to make it playable both on the web and in a CLI. For the CLI version, I wanted to test that the program was printing the right things to stdout at the right times.

There's a point in the game when players can swap their cards for new ones. The function `offerCardSwap` does the following:

- Ask the player if they want to swap any of their cards
- Collect their answer (yes or no)
- Ask them to retry if the answer is invalid
- Assume the answer is "no" if the player doesn't respond in time

The goal is to test `offerCardSwap` by feeding it various inputs, just as a real player would.

Let's walk through how to make `offerCardSwap` testable. Here's the first version of the code, which is tricky to test:

```go
func offerCardSwap() bool {
	timeout := time.Duration(30 * time.Second)

	message := "Would you like to swap any of your cards? [y/n]"
	retryMessage := "Invalid input"

	inputChan := make(chan bool)

	go func(inputChan chan bool) {
		reader := bufio.NewScanner(os.Stdin)

		var validResponse, response bool
		for !validResponse {
			fmt.Println(message)
			reader.Scan()

			switch userInput {
				case "Y","y":
					response, validResponse := true, true
				case "N","n":
					validResponse := true
				default:
					fmt.Println(retryMessage)
			}
		}

		inputChan <- response
	}(inputChan)

	select {
	case choice := <-input:
		return choice

	case <-time.After(timeout):
		return false
	}
}
```

The function does the following:
- Kicks off a goroutine that prints a message to stdout and waits for input from stdin
- Checks the input is valid and asks the player to retry if it's not
- If a valid response is received, the `for` loop breaks and the response is sent on the `inputChan` channel
- The `select` statement returns either the value from `inputChan` or a default value after the timeout elapses

If you called `offerCardSwap` in a test suite, you would see the message print to the command line, which you can't test. The tests would also hang for the length of the timeout, which is 30 seconds. Not ideal.

To make `offerCardSwap` testable, these problems must be solved:
- In tests, divert the messages to the player to somewhere other than stdin and simulate their response from somewhere other than stdout
- Continue to use real stdin and stdout in the main application
- Prevent tests from hanging for 30 seconds

These can be solved with dependency injection, which allows us to choose what to pass in at runtime.

### Capturing data sent to stdout
We want the output to go to stdout in the application, but not in tests.

We replace `fmt.Println(message)` with [`fmt.Fprint(os.Stdout, message)`][Fprint], which has the exact same behaviour. The difference is that `os.Stdout` is passed in explicitly, which allows us to swap it for something else when testing. 

Same for `fmt.Println(retryMessage)`, which becomes `fmt.Fprint(os.Stdout, retryMessage)`

```go
func offerCardSwap(w io.Writer) bool {
		// ... previous code

		for !validResponse {
			fmt.Fprint(w, message) // changed

			// ...

				default:
					fmt.Fprint(w, retryMessage) // changed
			}
		}
		ch <- response
	}(inputChan)

	// rest of code ...
}
```
```go
// somewhere in the application
shouldSwapCards := offerCardSwap(os.Stdin)
```

Since `fmt.Fprint` takes anything that satisfies the `io.Writer` interface, we can inject `*bytes.Buffer`. Messages sent to the player will end up in the buffer, which we can inspect.

Now it's possible to test the message that `offerCardSwap` displays to the player:
```go
func TestOfferCardSwap(t *testing.T) {
	t.Run("player sees message", func(t *testing.T) {
		notStdout := &bytes.Buffer{}
		offerCardSwap(notStdout)

		got := notStdout.String()
		want := "Would you like to swap any of your cards? [y/n]"

		if got != want {
			t.Errorf("got %s, want %s", got, want)
		}
	})
}
```


### Simulating input from stdin
 
[`bufio.NewScanner`][NewScanner] is responsible for reading in user input. To pass in `os.Stdin` explicitly, we define an `io.Writer` parameter.

```go
func offerCardSwap(r io.Reader, w io.Writer) { // changed
		// ... previous code

		for !validResponse {
			fmt.Fprint(w, message) // changed

		// rest of code ...
}
```

```go
// somewhere in the application
shouldSwapCards := offerCardSwap(os.Stdin, os.Stdout)
```

In the tests, we pass [`strings.NewReader`][NewReader], which satisfies the `io.Reader` interface. 

Now we can test a variety of inputs that the player could submit:
```go
func TestOfferCardSwap(t *testing.T) {
	t.Run("'yes' inputs", func(t *testing.T) {
		yesCases := []struct {
			input string
			want  bool
		}{
			{"y", true},
			{"Y", true},
			{"ja", false},
			{"oui", false},
			{"sure", false},
		}

		for _, c := range yesCases {
			notStdout := &bytes.Buffer{}
			notStdin := strings.NewReader(c.input)

			got := offerCardSwap(notStdout, notStdin)

			if got != c.want {
				t.Errorf("got %s, want %s", got, c.want)
			}
		}
	})
}
```

### Configuring the timeout
Waiting 30 seconds is fine for production, but we don't want to wait that long in tests. Let's inject that too.
```go
func offerCardSwap(r io.Reader, w io.Writer, inputTimeout time.Duration) { // changed
	// rest of code ...
}
```
```go
// somewhere in the application
const inputTimeout = time.Duration(30 * time.Second)

shouldSwapCards := offerCardSwap(os.Stdin, os.Stdout, inputTimeout)
```
This allows us to define a shorter duration in the tests:
```go
func TestOfferCardSwap(t *testing.T) {
	testTimeout := time.Duration(100 * time.Millisecond)

	t.Run("player sees message", func(t *testing.T) {
		notStdout := &bytes.Buffer{}
		notStdin := strings.NewReader("")
		offerCardSwap(notStdIn, notStdout, testTimeout) // changed

		got := notStdout.String()
		want := "Would you like to swap any of your cards? [y/n]"

		if got != want {
			t.Errorf("got %s, want %s", got, want)
		}
	})
}
```

### Tidy up
Almost there. Let's define `message` and `retryMessage` outside of `offerCardSwap` and inject those as well. Easier to maintain if we want to change the text in future.
```go
// somewhere in the application
const (
	message		 = "Would you like to swap any of your cards? [y/n]"
	retryMessage = "Invalid input"
	inputTimeout = time.Duration(30 * time.Second)
)

shouldSwapCards := offerCardSwap(os.Stdin, os.Stdout, inputTimeout, message, retryMessage)
```

Here's the final look for `offerCardSwap`:
```go
func offerCardSwap(r io.Reader, w io.Writer, timeout time.Duration, message, retryMessage string) bool {
	inputChan := make(chan bool)

	go func(inputChan chan bool) {
		reader := bufio.NewScanner(r)

		var validResponse, response bool
		for !validResponse {
			fmt.Fprint(w, message)
			reader.Scan()

			switch userInput {
				case "Y","y":
					response, validResponse := true, true
				case "N","n":
					validResponse := true
				default:
					fmt.Fprint(w, retryMessage)
			}
		}

		inputChan <- response
	}(inputChan)

	select {
	case choice := <-input:
		return choice

	case <-time.After(timeout):
		return false
	}
}
```



[Shed]: https://github.com/minaorangina/shed
[NewScanner]: https://pkg.go.dev/bufio#NewScanner
[NewReader]: https://pkg.go.dev/bufio#NewReader
[Fprint]: https://pkg.go.dev/fmt#Fprint