## Integration testing with a separate web server

The goal here is to:

1. Spawn a web server with a random port. 
2. Send requests against this new web server. 
3. Assert.

This requires having 2 processes: 1 for the webserver and 1 for running queries against the webserver. Sounds like a need for parallelism or at least concurrency has arisen. Before we go head first into goroutines, let's survey the ecosystem to see how this is being done.