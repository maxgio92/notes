---
Title: How I improved consistency in web crawling with Go
---

# Introduction

[wfind](https://github.com/maxgio92/wfind) is a simple web crawler for files and folders in web HTML pages. The goal is basically the same of [GNU find](https://www.gnu.org/software/findutils/manual/html_mono/find.html) for file systems.
At the same time it's inspired by [GNU wget](https://www.gnu.org/software/wget/manual/html_node/index.html), and it merges the `find` features applied to the web world.

> `TODO`

## ..And issues arose

As wfind leverages [go-colly](https://go-colly.org/) for the scraping of web HTML pages, it also enables the user to run it [asynchronously](https://go-colly.org/docs/examples/parallel/).
What that means is that it provides a way to execute the [`fetching`](https://github.com/gocolly/colly/blob/v2.1.0/colly.go#L440) of HTTP objects as [parallel work](https://github.com/gocolly/colly/blob/v2.1.0/colly.go#L573).

In the `wfind` implementation, the synchronization is as simple as invoking the [`Wait`](https://github.com/gocolly/colly/blob/v2.1.0/colly.go#L812) function, provided by the Colly collector, that wraps around the standard [`WaitGroup`](https://pkg.go.dev/sync#WaitGroup)'s `Wait()`, from the Go standard library's sync package.

As the go-colly implementation does not provide cap on the parallelism, the implementation can lead to the usual concurrency problems.
Client-side, the maxmimum allowed open connections could prevent the client to open and then establish new ones during the scraping.
The server could introduce problems and we cannot predict al lthe logics that could lead to them.
Also, the connection mean is another point of failure; for example latency might cause the HTTP client to time out during go-colly's [`Visit`](https://github.com/gocolly/colly/blob/v2.1.0/colly.go#L440C20-L440C27) waiting for a response.

At the end of the day a retry logics was fundamental in order to improve the consistency in the crawling.

Supporting the expected behaviour and result as a consumer, providing the tool as a black-box, end-to-end tests have been vital.

## The end-to-end tests

As end-to-end functional tests treat the program as a black-box and ensures that provide the value as expected, interacting with the real actors in the expected scenarios, I wrote tests again real CentOS kernel.org mirrors, looking for repository metadata files, as an example use case of `wfind`.

Knowing in advance the expected result the test would simply consider two main run mode: sync and async.
I used GinkGo as I like how it enables to provide readable tests and expected behaviour, for black-box tests in general - the same applies to integration and black-box unit tests to declare clearly the feature or unit specifications.

```go
package find_test

import (
	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"

	"github.com/maxgio92/wfind/internal/network"
	"github.com/maxgio92/wfind/pkg/find"
)

const (
	seedURL         = "https://mirrors.edge.kernel.org/centos/8-stream"
	fileRegexp      = "repomd.xml$"
	expectedResults = 155
)

var _ = Describe("File crawling", func() {
	Context("Async", func() {
		var (
			search = find.NewFind(
				find.WithAsync(true),
				find.WithSeedURLs([]string{seedURL}),
				find.WithFilenameRegexp(fileRegexp),
				find.WithFileType(find.FileTypeReg),
				find.WithRecursive(true),
			)
			actual        *find.Result
			err           error
			expectedCount = expectedResults
		)
		BeforeEach(func() {
			actual, err = search.Find()
		})
		It("Should not fail", func() {
			Expect(err).To(BeNil())
		})
		It("Should stage results", func() {
			Expect(actual.URLs).ToNot(BeEmpty())
			Expect(actual.URLs).ToNot(BeNil())
		})
		It("Should stage exact result count", func() {
			Expect(len(actual.URLs)).To(Equal(expectedCount))
		})
	})
	Context("Sync", func() {
		[...]
  })
})
```

As you can note, the order in which results are returned is not important and thus not tested.

## Retry logics

The first concrete goal of the retry logics was to start to see green flags from the GinkGo output:

```shell
$ ginkgo --focus "File crawling" pkg/find
[...]
FAIL! [...]
```

A mean to ensure, that requests failed would have been retried was essential.
Fortunately go-colly provide way to register a callback, that as per the documentation it registers a function that will be executed if an error occurs during the HTTP request, with [`OnError`](https://github.com/gocolly/colly/blob/v2.1.0/colly.go#L917).

That way it's possible to run a custom handler as the response (and the request) object and the error are available in context of the helper, as for the [signature](https://github.com/gocolly/colly/blob/v2.1.0/colly.go#L146).

### Dumb retrier

The first implementation of the retry could have been as simple as retry for a fixed amount of times, after a fixed amount of period.

For example:

```go
collector.OnError(func(resp *colly.Response, err error) {
  time.Sleep(2 * time.Second)
  resp.Request.Retry()
})
```

For sure this wasn't enough.

### Retry with exponential backoff

At first, a single retry might not be enough, and also, a fixed backoff could be too big or too small period depending on the type of failure and the context.

So I decided to leverage the community projects and digging around backoff implementations. After that, I picked and imported [`github.com/cenkalti/backoff`](https://github.com/cenkalti/backoff) package.
I liked the design as it respects all the SOLID principles. As that, it allows me to mix and match with further custom backoff implementations, still leveraging its tickers.

Furthermore, I wanted to provide knobs to control retry for specific errors encountered during the HTTP request. So I ended up with something like this:

```go
// handleError handles an error received making a colly.Request.
// It accepts a colly.Response and the error.
func (o *Options) handleError(response *colly.Response, err error) {
	switch {
	// Context timed out.
	case errors.Is(err, context.DeadlineExceeded):
		if o.ContextDeadlineExceededRetryBackOff != nil {
			retryWithExponentialBackoff(response.Request.Retry, o.TimeoutRetryBackOff)
		}
	// Request has timed out.
	case os.IsTimeout(err):
		if o.TimeoutRetryBackOff != nil {
			retryWithExponentialBackoff(response.Request.Retry, o.TimeoutRetryBackOff)
		}
	// Connection has been reset (RST) by the peer.
	case errors.Is(err, unix.ECONNRESET):
		if o.ConnResetRetryBackOff != nil {
			retryWithExponentialBackoff(response.Request.Retry, o.ConnResetRetryBackOff)
		}
	// Other failures.
	default:
	}
}
```

With the implementation of the retry leveraging the cenkalti's `backoff` package as for its official [example](https://github.com/cenkalti/backoff/blob/v4/example_test.go#L42C1-L71C2):

```go
// retryWithExtponentialBackoff retries with an exponential backoff a function.
// Exponential backoff can be tuned with options accepted as arguments to the function.
func retryWithExponentialBackoff(retryF func() error, opts *ExponentialBackOffOptions) {
	ticker := backoff.NewTicker(
		utils.NewExponentialBackOff(
			utils.WithClock(opts.Clock),
			utils.WithInitialInterval(opts.InitialInterval),
			utils.WithMaxInterval(opts.MaxInterval),
			utils.WithMaxElapsedTime(opts.MaxElapsedTime),
		),
	)

	var err error

	// Ticks will continue to arrive when the previous retryF is still running,
	// so operations that take a while to fail could run in quick succession.
	for range ticker.C {
		if err = retryF(); err != nil {
			// Retry.
			continue
		}

		ticker.Stop()
		break
	}

	if err != nil {
		// Retry has failed.
		return
	}

	// Retry is successful.
}
```

The end-to-end test could have been then updated with something like:

```go
var _ = Describe("File crawling", func() {
	Context("Async", func() {
		var (
			search = find.NewFind(
				find.WithAsync(true),
				find.WithSeedURLs([]string{seedURL}),
				find.WithClientTransport(network.DefaultClientTransport),
				find.WithFilenameRegexp(fileRegexp),
				find.WithFileType(find.FileTypeReg),
				find.WithRecursive(true),

				// Enable retry backoff with default parameters.
				find.WithContextDeadlineExceededRetryBackOff(find.DefaultExponentialBackOffOptions),
				find.WithConnTimeoutRetryBackOff(find.DefaultExponentialBackOffOptions),
				find.WithConnResetRetryBackOff(find.DefaultExponentialBackOffOptions),
			)
			actual        *find.Result
			err           error
			expectedCount = expectedResults
		)
		BeforeEach(func() {
			actual, err = search.Find()
		})
		It("Should not fail", func() {
			Expect(err).To(BeNil())
		})
		It("Should stage results", func() {
			Expect(actual.URLs).ToNot(BeEmpty())
			Expect(actual.URLs).ToNot(BeNil())
		})
		It("Should stage exact result count", func() {
			Expect(len(actual.URLs)).To(Equal(expectedCount))
		})
	})
})
```

being the retry options inherited by the previous `retryWithExponentialBackoff` function.

You can read here all the `find` options, including the new retry ones:
```go
// Options represents the options for the Find job.
type Options struct {
	// SeedURLs are the URLs used as root URLs from which for the find's web scraping.
	SeedURLs []string

	// FilenameRegexp is a regular expression for which a pattern should match the file names in the Result.
	FilenameRegexp string

	// FileType is the file type for which the Find job examines the web hierarchy.
	FileType string

	// Recursive enables the Find job to examine files referenced to by the seeds files recursively.
	Recursive bool

	// Verbose enables the Find job verbosity printing every visited URL.
	Verbose bool

	// Async represetns the option to scrape with multiple asynchronous coroutines.
	Async bool

	// ConnResetRetryBackOff controls the error handling on responses.
	// If not nil, when the connection is reset by the peer (TCP RST), the request
	// is retried with an exponential backoff interval.
	ConnResetRetryBackOff *ExponentialBackOffOptions

	// TimeoutRetryBackOff controls the error handling on responses.
	// If not nil, when the connection times out (based on client timeout), the request
	// is retried with an exponential backoff interval.
	TimeoutRetryBackOff *ExponentialBackOffOptions

	// ContextDeadlineExceededRetryBackOff controls the error handling on responses.
	// If not nil, when the request context deadline exceeds, the request
	// is retried with an exponential backoff interval.
	ContextDeadlineExceededRetryBackOff *ExponentialBackOffOptions
}
```

And now let's run the e2e test again:

```shell
$ ginkgo --focus "File crawling" pkg/find
OOM killed
```

It was likely that a memory leak or high consumption was already present, but without neither performance tests nor retry logics, no problems arose.

So, a heap memory profile was then needed. 

### Entering pprof

Long story short, pprof is a standard library's package that serves via its HTTP server runtime profiling data in the format expected by the pprof visualization tool.

> I recommend the official documentation of the package, and this great [Julia Evans' blog](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/).

So, I simply linked pprof package:

```go
package find

import (
  [...]
  _ "net/http/pprof"

  [...]
)
```

and modified the tested function `Find()` to run its webserver in parallel:

```go
func (o *Options) Find() (*Result, error) {
	go func() {
		log.Println(http.ListenAndServe("localhost:6060", nil))
	}()

	if err := o.Validate(); err != nil {
		return nil, errors.Wrap(err, "error validating find options")
	}

	switch o.FileType {
	case FileTypeReg:
		return o.crawlFiles()
	case FileTypeDir:
		return o.crawlFolders()
	default:
		return o.crawlFiles()
	}
}
```

run the tests:
```shell
$ ginkgo --focus "File crawling" pkg/find
```

and immediately invoke the pprof go tool to download the heap memory profile as a PNG image:

```shell
$ go tool pprof http://localhost:6060/debug/pprof/heap
(pprof) png
Generating report in profile001.png
```

Looking at the graph it was evident that the great amount of memory mapping was request by a buffer reading:`io.ReadAll()`, called from `colly(*httpBackend).Do()`:

![image](https://github.com/maxgio92/notes/assets/7593929/0e6da0f0-929d-456c-bc6f-ce5300750265)

So digging into the go-colly HTTP backend `Do` implementation, the [offending line](https://github.com/gocolly/colly/blob/v2.1.0/http_backend.go#L209) was:

```go
func (h *httpBackend) Do(request *http.Request, bodySize int, checkHeadersFunc checkHeadersFunc) (*Response, error) {
	...

	res, err := h.Client.Do(request)
	if err != nil {
		return nil, err
	}
	defer res.Body.Close()
	...

	var bodyReader io.Reader = res.Body
	if bodySize > 0 {
		bodyReader = io.LimitReader(bodyReader, int64(bodySize))
	}
	...
	body, err := ioutil.ReadAll(bodyReader)
	...
}
```

So, a mean to limit response body was mandatory.

### Max HTTP body size

Fortunately, go-colly provides a way to set the requests' max body size, so I ended up exposing a knob in the `find` functional options:

```go
package find

...

// NewFind returns a new Find object to find files over HTTP and HTTPS.
func NewFind(opts ...Option) *Options {
	o := &Options{}

	for _, f := range opts {
		f(o)
	}

	o.init()

	return o
}

func WithMaxBodySize(maxBodySize int) Option {
	return func(opts *Options) {
		opts.MaxBodySize = maxBodySize
	}
}
```

```go
// Options represents the options for the Find job.
type Options struct {
	...

	// ClientTransport represents the Transport used for the HTTP client.
	ClientTransport http.RoundTripper

	// MaxBodySize is the limit in bytes of each of the retrieved response body.
	MaxBodySize int

	...
}
```

which then would have fill the go-colly setting:

```go
package find

...

// crawlFiles returns a list of file names found from the seed URL, filtered by file name regex.
func (o *Options) crawlFiles() (*Result, error) {
	...

	// Create the collector settings
	coOptions := []func(*colly.Collector){
		...
		colly.MaxBodySize(o.MaxBodySize),
	}
```

and updated the end-to-end test, tuning the parameter with an expected maximum value:

```go
var _ = Describe("File crawling", func() {
	Context("Async", func() {
		var (
			search = find.NewFind(
				find.WithAsync(true),
				find.WithSeedURLs([]string{seedURL}),
				find.WithFilenameRegexp(fileRegexp),
				find.WithFileType(find.FileTypeReg),
				find.WithRecursive(true),
				find.WithMaxBodySize(1024*512),
				find.WithConnTimeoutRetryBackOff(find.DefaultExponentialBackOffOptions),
				find.WithConnResetRetryBackOff(find.DefaultExponentialBackOffOptions),
			)
			actual        *find.Result
			err           error
			expectedCount = expectedResults
		)
		BeforeEach(func() {
			actual, err = search.Find()
		})
		It("Should not fail", func() {
			Expect(err).To(BeNil())
		})
		It("Should stage results", func() {
			Expect(actual.URLs).ToNot(BeEmpty())
			Expect(actual.URLs).ToNot(BeNil())
		})
		It("Should stage exact result count", func() {
			Expect(len(actual.URLs)).To(Equal(expectedCount))
		})
	})
	Context("Sync", func() {
		...
	})
}
```

run again the tests:

```shell
$ ginkgo --focus "File crawling" pkg/find
...
Ran 3 of 3 Specs in 7.552 seconds
SUCCESS! -- 3 Passed | 0 Failed | 0 Pending | 0 Skipped
```

and tests passed.

### More tuning: HTTP client's Transport

Another important network and connection parameters are provided with the go `Transport`.
Connection timeout, TCP keep alive interval, TLS handshake timeout, Go net/http idle connnection pool maximum size, idle connections timeout are just some of them.

The [connection pool](https://github.com/golang/go/blob/go1.21.0/src/net/http/transport.go#L925) size here is fundamental to be tuned in order to satisfy the level of concurrency enabled by the asynchronous mode of go-colly, hence of wfind.

In detail, Go net/http `Get` keeps the connection pool as a cache of TCP connections, but when all are in use it opens another one.
If the parallelism is greater than the limit of idle connections, the program is going to be regularly discarding connections and opening new ones, the latters ending up in `TIME_WAIT` TCP state for two minutes, tying up that connection.

> About `TIME_WAIT` TCP state I recommend [this blog](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux) by Vincent Bernat.

From the Go standard library net/http package:

```go
package http

...
type Transport struct {
	...

	// MaxIdleConns controls the maximum number of idle (keep-alive)
	// connections across all hosts. Zero means no limit.
	MaxIdleConns int

	// MaxIdleConnsPerHost, if non-zero, controls the maximum idle
	// (keep-alive) connections to keep per-host. If zero,
	// DefaultMaxIdleConnsPerHost is used.
	MaxIdleConnsPerHost int
```

As so, it was very useful to provide way to inject a Transport configured for specific use cases.

Long story short:

```go
package find

// Options represents the options for the Find job.
type Options struct {
	...

	// ClientTransport represents the Transport used for the HTTP client.
	ClientTransport http.RoundTripper

	...
}
```

and in the go-colly collector set-up:

```go
package find

// crawlFiles returns a list of file names found from the seed URL, filtered by file name regex.
func (o *Options) crawlFiles() (*Result, error) {
	...

	// Create the collector settings
	coOptions := []func(*colly.Collector){
		colly.AllowedDomains(allowedDomains...),
		colly.Async(o.Async),
		colly.MaxBodySize(o.MaxBodySize),
	}

	...

	// Create the collector.
	co := colly.NewCollector(coOptions...)
	if o.ClientTransport != nil {
		co.WithTransport(o.ClientTransport)
	}
```

Furthermore, from the `wfind` main command user perspective, the command `Run` would consume it as so:

```go
func (o *Command) Run(_ *cobra.Command, args []string) error {
	...

	// Network client dialer.
	dialer := network.NewDialer(
		network.WithTimeout(o.ConnectionTimeout),
		network.WithKeepAlive(o.KeepAliveInterval),
	)

	// HTTP client transport.
	transport := network.NewTransport(
		network.WithDialer(dialer),
		network.WithIdleConnsTimeout(o.IdleConnTimeout),
		network.WithTLSHandshakeTimeout(o.TLSHandshakeTimeout),
		network.WithMaxIdleConns(o.ConnPoolSize),
		network.WithMaxIdleConnsPerHost(o.ConnPoolPerHostSize),
	)

	// Wfind finder.
	finder := find.NewFind(
		find.WithSeedURLs(o.SeedURLs),
		find.WithFilenameRegexp(o.FilenameRegexp),
		find.WithFileType(o.FileType),
		find.WithRecursive(o.Recursive),
		find.WithVerbosity(o.Verbose),
		find.WithAsync(o.Async),
		find.WithClientTransport(transport),
	)
```

for which default command's flag default values are provided by wfind for its specific use case:

```go
package network

...

const (
	// DefaultTimeout is the default HTTP client transport's timeout in
	// milliseconds.
	// By default, is set to 180 seconds.
	DefaultTimeout = 180000

	// DefaultKeepAlive is the default interval in milliseconds between
	// keep-alive probes for an active network connection.
	// By defdault, is set to 30 seconds.
	DefaultKeepAlive = 30000

	// DefaultMaxIdleConns is the default maximum number of idle
	// (keep-alive) connections across all hosts.
	// By default, is set to 1000.
	DefaultMaxIdleConns = 1000

	// DefaultMaxIdleConnsPerHost is the default maximum number of idle
	// (keep-alive) connections to keep per-host.
	// By default, is set to 1000.
	DefaultMaxIdleConnsPerHost = 1000

	// DefaultIdleConnTimeout is the default maximum amount of time in milliseconds an
	// idle (keep-alive) connection will remain idle before closing itself.
	// By default, is set to 120 seconds.
	DefaultIdleConnTimeout = 120000

	// DefaultTLSHandshakeTimeout is the default maximum amount of time waiting to
	// wait for a TLS handshake.
	// By default, is set to 30 seconds.
	DefaultTLSHandshakeTimeout = 30000
)
```

## Conclusion

`TODO`
