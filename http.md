# Golang notes: http

The simplest http server can be created and served via the `net/http` package

```go
func main() {
	http.ListenAndServe(":8080", nil)
}
```

>ListenAndServe listens on the TCP network address (addr) and then calls Serve with (type) Handler to handle requests on incoming connections. Accepted connections are configured to enable TCP keep-alives. The handler is typically nil, in which case the DefaultServeMux is used.

ListenAndServe always returns a non-nil error.

From the above, we see that the first argument accepted is an address to listen on, in this case `":8080"` which instructs the server to listen on all IPs at port :8080. The second argument (passed in as nil) is a handler, but since we do not pass any instance of a Handler, the DefaultServeMux is used instead.

As proof that this is listening, we can send start the server and curl it:

```bash
$ curl -v localhost:8080
```

```
*   Trying ::1:8080...
* Connected to localhost (::1) port 8080 (#0)
> GET / HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.73.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 404 Not Found
< Content-Type: text/plain; charset=utf-8
< X-Content-Type-Options: nosniff
< Date: Mon, 02 Nov 2020 03:40:53 GMT
< Content-Length: 19
<
404 page not found
* Connection #0 to host localhost left intact
```

Nothing much exciting happened, but that is because we do not have any handlers on the serve multiplexer to handle our incoming requests. We can do so by attaching a `HandlerFunc`
```go
func http.HandleFunc(pattern string, handler func(http.ResponseWriter, *http.Request))
```

> HandleFunc registers the handler function for the given pattern in the DefaultServeMux. The documentation for ServeMux explains how patterns are matched.

So we can pass a string, for a pattern, and a callback of type handler to... handle... the request.

To respond the the request, we'll be using the `http.ResponseWriter` type, if we check the documentation we will see the `Write` method is available to us with the following signature:

```go
Write([]byte) (int, error)
```

Let's try that:

```go
func main() {
	http.HandleFunc("/", func(rw http.ResponseWriter, r *http.Request) {
		rw.Write([]byte("Hello, world!\r\n"))
	})

	http.ListenAndServe(":8080", nil)
}
```

And then again with a curl:

```bash
curl -v localhost:8080
```

```
*   Trying ::1:8080...
...
* Connection #0 to host localhost left intact
Hello, world!
```

Reading from the body is similarly quite simple. If we inspect the type of the `http.Request` we can see the Body is of type `io.ReadCloser` which means it implements the std/io libaries. We can use `ioutil.ReadAll` to capture some data from our curl and then pass it along to our ResponseWriter and back to the user:

```go
func ioutil.ReadAll(r io.Reader) ([]byte, error)
```

>ReadAll reads from r until an error or EOF and returns the data it read. A successful call returns err == nil, not err == EOF. Because ReadAll is defined to read from src until EOF, it does not treat an EOF from Read as an error to be reported.

Notice that the `ReadAll` method returns a byte slice as well as an error. It is good practice to catch the err from all throwable method invocations:

```go
if err != nil {
	// Handle the error
}
```

Recall again that `ListenAndServe` can also return an error, so we will catch both, and our updated code should look as follows:

```go
func main() {
	http.HandleFunc("/", func(rw http.ResponseWriter, r *http.Request) {
		body, err := ioutil.ReadAll(r.Body)
		if err != nil {
			fmt.Printf("Error reading the request, %s", err.Error())
		}

		rw.Write([]byte(fmt.Sprintf("Hello, %s", body)))
	})

	err := http.ListenAndServe(":8080", nil)
	if err != nil {
		log.Fatal(err)
	}
}
```

```bash
curl -d "Discord!" localhost:8080
```

and our response:
```
Hello, Discord!
```
