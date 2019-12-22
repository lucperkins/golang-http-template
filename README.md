# OpenFaaS Golang HTTP templates

This repository contains two Golang templates for OpenFaaS which give additional control over the HTTP request and response. They will both handle higher throughput than the classic watchdog due to the process being kept warm.

```
$ faas template pull https://github.com/openfaas-incubator/golang-http-template
$ faas new --list

Languages available as templates:
- golang-clooudevents
- golang-http
- golang-http-armhf
- golang-middleware
- golang-middleware-armhf
```

> To build and deploy a function for your Raspberry Pi or ARMv7 in general, use the language templates ending in `-armhf`.

The two templates are equivalent with `golang-http` using a structured request/response object and the alternative implementing a Golang `http.HandleFunc` from the Golang stdlib. `golang-http` is more "conventional" for a Golang serverless template but this is a question of style/taste.

## 1.0 golang-http

This template provides additional context and control over the HTTP response from your function.

### Status of the template

This template is pre-release and is likely to change - please provide feedback via https://github.com/openfaas/faas

The template makes use of the OpenFaaS incubator project [of-watchdog](https://github.com/openfaas-incubator/of-watchdog).

### Trying the template

```
$ faas template pull https://github.com/openfaas-incubator/golang-http-template
$ faas new --lang golang-http <fn-name>
```

### Example usage

Example writing a successful message:

```go
package function

import (
	"fmt"
	"net/http"

	"github.com/openfaas-incubator/go-function-sdk"
)

// Handle a function invocation
func Handle(req handler.Request) (handler.Response, error) {
	var err error

	message := fmt.Sprintf("Hello world, input was: %s", string(req.Body))

	return handler.Response{
		Body:       []byte(message),
    }, err
}
```

Example writing a custom status code

```go
package function

import (
	"fmt"
	"net/http"

	"github.com/openfaas-incubator/go-function-sdk"
)

// Handle a function invocation
func Handle(req handler.Request) (handler.Response, error) {
	var err error

	return handler.Response{
		Body:       []byte("Your workload was accepted"),
		StatusCode: http.StatusAccepted,
	}, err
}
```

Example writing an error / failure.

```go
package function

import (
	"fmt"
	"net/http"

	"github.com/openfaas-incubator/go-function-sdk"
)

// Handle a function invocation
func Handle(req handler.Request) (handler.Response, error) {
	var err error

	return handler.Response{
        Body: []byte("the input was invalid")
	}, fmt.Errorf("invalid input")
}
```

The error will be logged to `stderr` and the `body` will be written to the client along with a HTTP 500 status code.

Example reading a header.

```go
package function

import (
	"log"

	"github.com/openfaas-incubator/go-function-sdk"
)

// Handle a function invocation
func Handle(req handler.Request) (handler.Response, error) {
	var err error

	log.Println(req.Header) // Check function logs for the request headers

	return handler.Response{
		Body: []byte("This is the response"),
		Header: map[string][]string{
			"X-Served-By": []string{"My Awesome Function"},
		},
	}, err
}
```

## 2.0 golang-middleware

This template uses the [http.HandlerFunc](https://golang.org/pkg/net/http/#HandlerFunc) as entry point.

### Status of the template

The template makes use of the OpenFaaS incubator project [of-watchdog](https://github.com/openfaas-incubator/of-watchdog).

### Trying the template

```
$ faas template pull https://github.com/openfaas-incubator/golang-http-template
$ faas new --lang golang-middleware <fn-name>
```

### Example usage

Example writing a json response:

```go
package function

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
)

func Handle(w http.ResponseWriter, r *http.Request) {
	var input []byte

	if r.Body != nil {
		defer r.Body.Close()

		// read request payload
		reqBody, err := ioutil.ReadAll(r.Body)

		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return

		input = reqBody
		}
	}

	// log to stdout
	fmt.Printf("request body: %s", string(input))

	response := struct {
		Payload     string              `json:"payload"`
		Headers     map[string][]string `json:"headers"`
		Environment []string            `json:"environment"`
	}{
		Payload:     string(input),
		Headers:     r.Header,
		Environment: os.Environ(),
	}

	resBody, err := json.Marshal(response)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

    // write result
	w.WriteHeader(http.StatusOK)
	w.Write(resBody)
}
```

Example persistent database connection pool between function calls:

```go
package function

import (
	"database/sql"
	"fmt"
	"io/ioutil"
	"net/http"
	"strings"
	_ "github.com/go-sql-driver/mysql"
)

// db pool shared between function calls
var db *sql.DB

func init() {
	var err error
	db, err = sql.Open("mysql", "user:password@/dbname")
	if err != nil {
		panic(err.Error())
	}

	err = db.Ping()
	if err != nil {
		panic(err.Error())
	}
}

func Handle(w http.ResponseWriter, r *http.Request) {
	var query string

	if r.Body != nil {
		defer r.Body.Close()

		// read request payload
		body, err := ioutil.ReadAll(r.Body)

		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}

		query = string(body)
	}

	// log to stdout
	fmt.Printf("Executing query: %s", query)

	rows, err := db.Query(query)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	defer rows.Close()
	
	ids := make([]string, 0)
	for rows.Next() {
		var id int
		if err := rows.Scan(&id); err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		ids = append(ids, string(id))
	}
	if err := rows.Err(); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	
	result := fmt.Sprintf("ids %s", strings.Join(ids, ", "))

	// write result
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(result))
}
```

## golang-cloudevents

This template enables you to handle a [CloudEvents](https://cloudevents.io) [`Event`](https://godoc.org/github.com/cloudevents/sdk-go/pkg/cloudevents#Event) directly, without needing to derive the `Event` from the HTTP request.

Here's an example:

```go
package function

import (
		"log"

		"github.com/my-scooter/scooters" // Fictional library
		"github.com/clooudevents/sdk-go"
)

func Handle(event cloudevents.Event) {
		var data map[string]interface{}

		if err := event.DataAs(&data); err != nil {
				log.Printf("could not parse event data: %v", err)
		}

		scooterId := data["scooter"]["id"]

		switch t := event.Type() {
		case "scooter.unlock":
				scooters.Unlock(scooterId)
		case "scooter.lock":
				scooters.Lock(scooterId)
		case "scooter.recharge":
				scooters.Recharge(scooterId)
		default:
				log.Printf("did not recognize event type %s", t)
		}
}
```

As you can see, the CloudEvents template does *not* provide an HTTP response writer, which makes it more suitable for "trigger and forget" use cases than for use cases that require an HTTP response from the function.
