---
title: "CI Server"
date: 2018-01-05T09:17:31-06:00
draft: false
---

How to build a ci server in go
---

For my new blog, I wanted to have the same sort of developer experience as I do with most of my work projects. In most cases,
building and deploying are either automated or a single script. I didn't want to set up Jenkins or TeamCity or what have you.
I had a simpler set of requirements in mind.

My ideal workflow for a static blog would be: drafting content in markdown and pushing commits to git. Everything after that should be automated.

After thinking about this, I figured I could whip something together that does this easily. The basic idea is setting up a git webhook for pushes
and setting up a server to respond to them.

The server implementation should be simple:

* should listen to git pushes
* on each push, it should rebuild static content and push to an s3 bucket
* should support some basic diagnostics like listing deployments and seeing the output of each log

After a bit of hacking I came up with [boop](https://github.com/prestonp/boop). This blog will cover
how I built boop. The server was written in Go and hosted on a single DigitalOcean droplet.

The droplet contains several virtual hosts behind a nginx reverse proxy. This is all coordinated with
docker and https://github.com/jwilder/nginx-proxy.

### Setting up a webhook

In github you can configure web hooks on key events. A web hook is just an http request typically
with some interesting set of data.

<aside>
Sometimes services use hooks as a notification mechanism. One example might be uploading and encoding
a video file. This typically takes a long time to complete so most APIs serving this kind of work are asynchronous. Going off the
upload example, say a client makes an upload request. An async API would return immediately with an HTTP status code
202, indicating that the request was _received_, but not necessarily completed. 

If this was API was sync, the server would have to complete all the work first and _then_ respond. This could take minutes to hours depending
on the workload. Clearly not ideal. So, instead of having clients poll for state on async, clients could implement a hook to receive status updates.
</aside>

I created a new repo on github and configured a webhook. I set a secret which is used to digitally sign the webhook (HMAC).

So to start off building our ci server we'll need to _listen_ to webhooks. When a webhook request is received, we need to verify the HMAC.
Github specifies X-Hub-Signature header which is the HMAC hex digest of the payload body. To verify, just take the sha1 hash of the body with the aforementioned
HMAC secret. If the computed digest matches the header, then the webhook is legitimate.

This is pretty trivial to implement with crypto libs but I found a [lib](https://github.com/rjz/githubhook) that already handles this.

### Triger deployments

When a webhook handler is invoked, I run a simple bash script to build the static content then sync it up to a s3 bucket.

```
#! /bin/bash

set -x
rm -rf build

git clone --recursive git://github.com/prestonp/bloggo.git build

cd build

hugo

aws s3 sync ./public s3://blog.preston.io --region us-west-1
```

So, basically clone, build, push. As per one of the requirements, I wanted to be able to look back on logs in case
something goes wrong during `hugo` build. To do this, I set up logging out to a log file.

### API

The API only supports two operations:

`GET /` - List all deployments as text/html

`GET /log/:idx` - Fetch a specific log where :idx is the deployment number

There's a small bit of state that is recorded in-memory for all of the deployments. Things like when the 
deployment started and ended, the status of the deployment (inprogress, success or fail).

To simplify the implementation I made the decision to store all this state in-memory. Restarting the service cleans the logs and metadata, 
but I didn't need this to be a robust or scalable service. It would hardly see any traffic because it only needs to responds to changes I make on a repo. 
So in-memory is completely fine.

Visit https://ci.preston.io to see it in action.

### Configuration

The rest of the details are described in the Dockerfile and configuration is passed through
`docker run` environment variables. Most importantly you'll need your AWS credentials and your
GitHub HMAC secret.

Looking back on this, I wanted everything to be flexible and configurable but I ended up implementing
a very specific workflow (github -> ci -> s3). Abstracting out the different pieces to support alternative
triggers and would make this a more reusable project, but my original goal was simply automating my blog deployments. 

To that end, building the blog and uploading takes about ~3 seconds. Another advantage of this system is being able 
to just focus on content. As long as I have github access, I can just write markdown files and commit to create new entries. 
No need to have hugo and an aws client installed everywhere. Just push and you're done.
