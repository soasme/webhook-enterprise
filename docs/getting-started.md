# Getting Started

Webhook-Enterprise runs as a standalone application. It provides out-of-box
experience, yet powerful to tune.

Eager to get started? Let's begin our journey! It assumes you have already
installed Webhook-Enterprise in your system, otherwise, check [Installation]
section.

## A Minimal Configuration

A minimal configuration for running webhook-enterprise application looks
something similar like this:

```yaml
SECURE_KEY='vqFsudxydWYoFzxVOZQaWJtwTroOHZBECUyDEZzoxLQzOMlzyO'
```

So what did we just configured?

* `SECURE_KEY`

Save the code fragment above as `.env` in your current directory. To run
webhook-enterprise you can use `webhook-cli` command. It will automatically
find and load the dotenv file (`.env`) you just placed:

```bash
$ webhook-cli run api
```

This launches a process handling incoming webhook requests. Yet we haven't
had any webhook related configurations, so basically it responds webhook 404 as
default behavior. Although we can curl `http://0.0.0.0:4698/health` for its health
information:

```bash
$ curl http://0.0.0.0:4698/health
{"status": "running", "issues": []}
```

## Expose to the world

```bash
$ ngrok http 4698
```

[Deployment]

## Source and Target

Source and target are key concepts in webhook-enterprise system. Any service
sending HTTP requests to webhook-enterprise is source, and any service receiving
HTTP requests that webhook-enterprise forwards is target. Let's make it more
functional now by adding source and target! 🤘

Source: Github, set webhook.

```bash
$ webhook-cli source add --name='Github' --auth='ghook:nosecret' \
  --status-code=200 --content-type='application/json' --content='{"message": "ok"}'
https://40d40092ddfcd570645c97b50a4a4cf7:aba8deb917b097e55fb932401c4d0d5b@0.0.0.0:4698/1
```

Target: Example Application.

```bash
$ webhook-cli target add --name='GHook Example' --no-ssl --no-auth
  --url='http://0.0.0.0:5000/webhook'
```

## Testing Proxy

Send request to webhook-enterprise.

```bash
$ curl https://40d40092ddfcd570645c97b50a4a4cf7:aba8deb917b097e55fb932401c4d0d5b@0.0.0.0:4698/1 -X POST \
  -H"Content-Type: application/json" -d '{"data": {}}'
```

We will see an source log. This means that we successfully received a request.

```bash
$ webhook-cli log --type source.request -n 1 --pretty
{
    "id": "c68db85fd7a47d6c3ac5d3bbe9742dac",
    "type": "source.request",
    "user-agent": "curl/7.51.0",
    "payload": {"data": {}},
    "created_at": "2017-05-14T02:23:50+0000",
    ...(trucated)
}
``

We will also see at least one target log. This means that we successfully forwarded
the request to target and got response from target.

```bash
$ webhook-cli log --type target.response -n 1 --pretty
{
    "id": "cbf67e4afcba55b93e04253628096f0d",
    "type": "target.response",
    "source": {
        "id": "c68db85fd7a47d6c3ac5d3bbe9742dac"
    },
    "response": {
        "status_code": 200,
    }
    ... (trucated)
}
```

## Retry

Target system might be down sometimes. Let's make our target system down for a while.

```python
$ sed -i 's/return 200, "OK"/return 500, "MAINTENANCE"' target.py
```

Flask application detects the modification and automatically reload itself. Now
all the requests sent to it will get response with 500 status code.

Let's send a request again:

```bash
$ curl https://40d40092ddfcd570645c97b50a4a4cf7:aba8deb917b097e55fb932401c4d0d5b@0.0.0.0:4698/1 -X POST \
  -H"Content-Type: application/json" -d '{"data": {}}'
```

As you might guess, webhook-enterprise will respond back with 200 status code.

```bash
$ webhook-cli log --type source.request -n 1 --pretty
{
    "id": "f6b9cf92a375205433bd0aa353f1864b",
    "type": "source.request",
    "user-agent": "curl/7.51.0",
    "payload": {"data": {}},
    "created_at": "2017-05-14T02:30:00+0000",
    ...(trucated)
}
```

But the target is in maintenance, so webhook-enterprise will run its retry policies.
There are several built-in retry policies. The default one is linear interval.
Let's view what's happening:

```bash
$ webhook-cli log -f --type target.response --pretty
{
    "id": "773b29af2cb07dd58e626f4e0c1c3dc2",
    "type": "target.response",
    "source": {
        "id": "c68db85fd7a47d6c3ac5d3bbe9742dac"
    },
    "response": {
        "status_code": 500,
    },
    "created_at": "2017-05-14T02:30:00+0000",
    "next_run_at": "2017-05-14T02:31:00+0000",
    ... (trucated)
}
{
    "id": "ab83fb20f20861aef970e4ede3cadfb5",
    "type": "target.response",
    "source": {
        "id": "c68db85fd7a47d6c3ac5d3bbe9742dac"
    },
    "response": {
        "status_code": 500,
    },
    "created_at": "2017-05-14T02:31:00+0000",
    "next_run_at": "2017-05-14T02:32:00+0000",
    ... (trucated)
}
```

It stops at last if target system keep respond with 500 as its status code.

## Resend

Resend the request is useful when your system is back to life.

Let's call our system awake:

```python
$ sed -i 's/return 500, "MAINTENANCE"/return 200, "OK"' target.py
```

Flask application reloads itself and respond 200 as status code again.

We can use `webhook-cli send` to resend stored messages.

```python
$ webhook-cli send --source-id "c68db85fd7a47d6c3ac5d3bbe9742dac"
```

This time we should get a successful response from target system:

```bash
$ webhook-cli log --type source.request -n 1 --pretty
{
    "id": "d3172ac3165389357a398383eea3faa9",
    "type": "source.request",
    "user-agent": "curl/7.51.0",
    "payload": {"data": {}},
    "created_at": "2017-05-14T02:30:00+0000",
    ...(trucated)
}
```

## Next Step

Now you have a glance at basic usage of webhook-enterprise, you will want
to explore more features of it.

* [Basic Concepts](/docs/basic-concepts.md)
* [Features](/docs/features.md)
* [Command Line Interfaces](/docs/cli.md)
