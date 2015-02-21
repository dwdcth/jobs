Jobs
====

A persistent and flexible background jobs library for go.

[![GoDoc](https://godoc.org/github.com/albrow/jobs?status.svg)](https://godoc.org/github.com/albrow/jobs)

Version: X.X.X

Supports the following features:

 - A job can encapsulate any arbitrary functionality. A job can do anything
   which can be done in a go function.
 - A job can be one-off (only executed once) or recurring (scheduled to
   execute at a specific interval).
 - A job can be retried a specified number of times if it fails.
 - A job is persistent, with protections against power loss and other worst
   case scenarios.
 - Jobs can be executed by any number of concurrent workers accross any
   number of machines, and each job will only be executed once.
 - You can query the database to find out e.g. the number of jobs that are
   currently executing or how long a particular job took to execute.
 - Any job that permanently fails will have its error captured and stored.


Why is it Useful?
-----------------

Jobs is intended to be used in web applications. It is useful for cases where you need
to execute some long-running code, but you don't want your users to wait for the code to
execute before rendering a response. A good example is sending a welcome email to your users
after they sign up. You can use Jobs to schedule the email to be sent asynchronously, and
render a response to your user without waiting for the email to be sent. You could use a
goroutine to accomplish the same thing, but in the event of a server restart or power loss,
the email might never be sent. Jobs guarantees that the email will be sent at some time,
and allows you to spread the work between different machines.


Quickstart Guide
----------------

### Registering Job Types

Jobs must be organized into discrete types. Here's an example of how to register a job
which sends a welcome email to users:

``` go
// We'll specify that we want the job to be retried 3 times before finally failing
welcomeEmailJobs, err := jobs.RegisterJobType("welcomeEmail", 3, func(user *User) {
	msg := fmt.Sprintf("Hello, %s! Thanks for signing up for foo.com.", user.Name)
	if err := emails.Send(user.EmailAddress, msg); err != nil {
		// Panics will be captured by a worker, triggering up to 3 retries
		panic(err)
	}
})
```

### Scheduling a Job

After registering a job type, you can schedule a job using the Schedule or ScheduleRecurring
methods like so:

``` go
// The priority argument lets you choose how important the job is. Higher
// priority jobs will be executed first.
job, err := welcomeEmailJobs.Schedule(100, time.Now(), &User{EmailAddress: "foo@example.com"})
if err != nil {
	// Handle err
}
```

You can use the [Job object](http://godoc.org/github.com/albrow/jobs#Job) returned by Schedule
or ScheduleRecurring to check on the status of the job or cancel it manually.

### Starting and Configuring Worker Pools

You can schedule any number of worker pools accross any number of machines, provided every machine
agrees on the definition of the job types. If you want, you can start a worker pool on the same
machines that are scheduling jobs, or you can have each worker pool running on a designated machine.
It is technically safe to start multiple pools on a single machine, but typically you should only start
one pool per machine.

To create a new pool with the [default configuration](http://godoc.org/github.com/albrow/jobs#pkg-variables),
just pass in nil:

``` go
pool := jobs.NewPool(nil)
```

You can also specify a different configuration by passing in
[*PoolConfig](http://godoc.org/github.com/albrow/jobs#PoolConfig). Any zero values in the config you pass
in will fallback to the default values. So here's how you could start a pool with 10 workers and a batch
size of 10, while letting the other options remain the default.

``` go
pool := jobs.NewPool(&jobs.PoolConfig{
	NumWorkers: 10,
	BatchSize: 10,
})
```

After you have created a pool, you can start it with the Start method. Once started, the pool will
continuously query the database for new jobs and delegate those jobs to workers. Any program that calls
Pool.Start() should also wait for the workers to finish before exiting. You can do so by wrapping Close and
Wait in a defer statement. Typical usage looks something like this:

``` go
func main() {
	pool := jobs.NewPool(nil)
	defer func() {
		pool.Close()
		if err := pool.Wait(); err != nil {
			// Handle err
		}
	}
	if err := pool.Start(); err != nil {
		// Handle err
	}
}
```

You can also call Close and Wait at any time to manually stop the pool from executing new jobs. In this
case, any jobs that are currently being executed will still finish.


License
-------

Jobs is licensed under the MIT License. See the LICENSE file for more information.
