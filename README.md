# AWS Lambda Debugger

Do you want to step through code running live in Lambda? Do you want to fix bugs faster?
Do you want free pizza?

This project will help you with the first 2 questions. When you show it to a friend,
you might get that 3rd one too :)

*This is only for the AWS Node 6.10 runtime*

## Isn't this impossible?

No. Well, not anymore.

## How?

Normally, debugging is a one hop process: developer's debugger connect directly to the process. This *is* impossible with Lambda.

However, we fork your code to a separate child process that is
running in debug mode and connected to the original via AN interprocess communication
channel. The parent process opens 2 WebSockets as well: one to the child process'
V8 Inspector and the other to a broker server, becoming a proxy between the 2
connections. Next, the developer connects a debugger to the broker server, which
connects them to the proxy, which is connected to the child's debugger port.
Now, you have a 3 hop chain like this:

```
Debugger <=> Broker <=> Proxy <=> Child
```

Once the developer's debugger connects to the broker server, the debug
version of the handler is executed. The proxy and the child coordinate to
shim the event, context, and callback. *Result:* the developer is connected
to a live running Lambda function with full control.

Oh, and you only have to add one line of code *at the end* of your handler file(s)...

## I want one!

Good. There are 5 steps:

1. Deploy the broker server
2. Add the proxy to your code
3. Configure the proxy via environment variables
4. Increase your Lambda timeout
5. Use it!

### Deploy the broker server

- Kick off an EC2 Amazon Linux Instance
- Attach Security Group
    - exposing ports `8080` and `9229` to `0.0.0.0/0`
    - expose port 22 to [YOUR IP](https://www.google.com/search?q=whats+my+ip)
- SSH in to the box

```bash
# Install Docker
sudo yum update -y
sudo yum install -y docker
sudo service docker start

# Run the Broker
docker run --name debug-broker \
    -d -p 8080:8080 -p 9229:9229 \
    --restart always \
    trek10/aws-lambda-debugger

# To view logs
docker logs -f debug-broker
```

### Add the proxy to your code

Add the package to your repo:

```bash
npm install aws-lambda-debugger --save
```

Require the package at the very end of each file that contains a Lambda handler
that you want to debug. Example:

```javascript
module.exports.myHandler = (event, context, callback) => {
  // put some code that you want to debug here
}

require('aws-lambda-debugger');
```

That's it!!!

### Configure the proxy via environment variables

There are 3 magic environment variables that need to be set:

- `DEBUGGER_ACTIVE`: As long as this value is present and it is not 'false'
or empty string, the proxy will do its job.
- `DEBUGGER_BROKER_ADDRESS`: This is the IP address or domain for the broker server.
- `DEBUGGER_FUNCTION_ID`: This is a unique ID (per function!) that is used to route the
debugger to the appropriate function. It is used as part of the URL to connect.

### Increase your Lambda timeout

This is pretty straight forward. Alter the timeout to 300 seconds to allow
maximum debug time.

### Use it!

1. Launch your Lambda function (from console, via CLI, etc)
2. Replace the `DEBUGGER_BROKER_ADDRESS` and `DEBUGGER_FUNCTION_ID` in the following URL
and paste it into Chrome.
```chrome-devtools://devtools/remote/serve_file/@60cd6e859b9f557d2312f5bf532f6aec5f284980/inspector.html?experiments=true&v8only=true&ws=[DEBUGGER_BROKER_ADDRESS]:9229/[DEBUGGER_FUNCTION_ID]```
3. DEBUG!!!

## What's the catch?

There are a few catches/known issues:

- Multiple `console.log` calls close together sometimes causes them to be
aggregated in a single CloudWatch Log entry
- This only works with the AWS Node 6.10 runtime. We expect it to work with
Node 8 whenever AWS offers support for it too.
- Chrome DevTools is the only debugger that we have proven to work. YMMV with
your IDE.
- You pay for your debug time. No way around this. It is a running Lambda
function after all.
- `context.callbackWaitsForEmptyEventLoop` is defaulted to `false`. You can
change it back to `true` (we even shimmed that!), but your function *will*
run until it times out if you do.
- `context.getRemainingTimeInMillis` is technically an approximation. We
grab the remaining time and the current timestamp when the debugger connects
and ship both to the child. The time delta is then subtracted from the original
time. Since all times are retrieved inside of the Lambda, this should be a
*very close* approximation.

## Anything else I should know?

Functionally, this thing is complete. However it is still very new,
so don't be surprised if something goes wrong. Feel free to file an issue.
We're happy to take PRs too.

## Items that need work still

- Tests: There are no tests. Because of all of the dark arts, eventing stuff,
and the sockets, we didn't want to delay release because the tests are going
to be hard to write. If you're a ninja at this, feel free to reach out.
- Improving logging in the broker server
- Bulletproof socket cleanup: We're still trying to make sure that all sockets
are cleaned up as fast as possible. If you find any leaks, please let us know.
- Study memory usage in the broker.

## Future ideas:

- Make things more configurable
- Web UI for broker server
  - Get direct link as soon as Lambda connects to broker
  - Track remaining time

**Made with :gift_heart: and :sparkles:magic:sparkles: by [Rob Ribeiro](https://github.com/azurelogic) + [Trek10](https://www.trek10.com/)**

P.S.: We do AWS consulting and monitoring. Come talk to us.

Twitter: [Rob](https://twitter.com/azurelogic) + [Trek10](https://twitter.com/trek10inc)