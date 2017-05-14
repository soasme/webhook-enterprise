# Getting Started

Webhook-Enterprise runs as a standalone application. It provides out-of-box
experience for being simple, and also plenty of options for being powerful.

Eager to get started? Let's begin our journey! It assumes you have already
installed Webhook-Enterprise in your system, otherwise, check [Installation]
section.

## Start an API Process

Webhook-Enterprise installs command `webhook-cli` in you OS. By running 
`webhook-cli run api`, you can launch Webhook-Enterprise api instance:

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

## Start an Worker

To enable forwarding the request, we need to run Webhook-Enterprise workers.
By running `webhook-cli run worker`, you can launch Webhook-Enterprise worker instance:

```bash
$ webhook-cli run worker
```

This launches a process consuming task sending jobs from message queue.

## Source, Target and Message

Source, target and message are key concepts in webhook-enterprise system.

* Source is any service sending HTTP requests to webhook-enterprise.
* Target is any service receiving HTTP requests from webhook-enterprise.
* Message is the information Webhook-Enterprise forward from source to target.

These concepts are explained thoroughly in [Basic Concepts].

Let's make it more functional now by adding a source and a target! 🤘

We add a source named 'Curl Example'. When Webhook-Enterprise received request,
it will respond immediately with 200 as status code, '{"message": "ok"}' as
JSON content.  CLI outputs the webhook url in the terminal.

```bash
$ webhook-cli source add --name='Curl Example' \
  --status-code=200 \
  --content-type='application/json' \
  --content='{"message": "ok"}'
https://40d40092ddfcd570645c97b50a4a4cf7:aba8deb917b097e55fb932401c4d0d5b@0.0.0.0:4698/1
```

As mentioned, we need a target service that Webhook-Enterprise forward the
message to. Let's prepare a real target service:

```bash
$ cd /tmp
$ cat > flask_app.py <<EOF
from flask import Flask

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    return 200, 'OK'

app.run(debug=True)
EOF

$ virtualenv venv && source venv/bin/activate && pip install flask
$ python flask_app.py
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
 * Restarting with stat
```

Now we've started a target service. Then we add a target named "Flask App".
It forwards messages from "Curl Example" to "Flask App" and sets the
webhook url of "Flask App" to "http://127.0.0.1:5000/webhook".

```bash
$ webhook-cli target add --name='Flask App' --forward-name "Curl Example"
  --url='http://127.0.0.1:5000/webhook'
```

## Testing Proxy

Below is a curl command posting JSON data to the url you just got when
adding source:

```bash
$ curl https://40d40092ddfcd570645c97b50a4a4cf7:aba8deb917b097e55fb932401c4d0d5b@0.0.0.0:4698/1 \
  -X POST -H"Content-Type: application/json" -d '{"data": {}}'
```

We will see an source request log. This means that we successfully received a request.

```bash
$ webhook-cli log --type message -n 1 --pretty
{
    "id": "c68db85fd7a47d6c3ac5d3bbe9742dac",
    "type": "message",
    "user-agent": "curl/7.51.0",
    "payload": {"data": {}},
    "created_at": "2017-05-14T02:23:50+0000",
    ...(trucated)
}
```

We will also see at least one target log. This means that we successfully forwarded
the message to target and got response from target.

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
$ sed -i '' "s/return 200, 'OK'/return 500, 'MAINTENANCE'/" flask_app.py
```

Flask application detects the modification and automatically reload itself. Now
all the requests sent to it will get response with 500 status code.

Let's send a request again:

```bash
$ curl https://40d40092ddfcd570645c97b50a4a4cf7:aba8deb917b097e55fb932401c4d0d5b@0.0.0.0:4698/1 -X POST \
  -H"Content-Type: application/json" -d '{"data": {}}'
```

As you might guess, webhook-enterprise will respond back with 200 status code.
But as the target is in maintenance, webhook-enterprise will run its retry policies.
There are several built-in retry policies. The default one is linear interval.
Let's view what's happening:

```bash
$ webhook-cli log --type message -n 1 --pretty
{
    "id": "f6b9cf92a375205433bd0aa353f1864b",
    "type": "message",
    "user-agent": "curl/7.51.0",
    "payload": {"data": {}},
    "created_at": "2017-05-14T02:30:00+0000",
    ...(trucated)
}
```

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

It stops at last if target system keep responding with 500 as its
status code.

## Resend

Resend the request is useful when your system is back to life.

Let's call our system awake:

```bash
$ sed -i '' "s/return 500, 'MAINTENANCE'/return 200, 'OK'/" flask_app.py
```

Flask application reloads itself and respond 200 as status code again.

We can use `webhook-cli send` to resend stored messages.

```bash
$ webhook-cli send --message-id "c68db85fd7a47d6c3ac5d3bbe9742dac"
```

This time we should get a successful response from target system:

```bash
$ webhook-cli log --type target.response -n 1 --pretty
{
    "id": "d3172ac3165389357a398383eea3faa9",
    "type": "target.response",
    "source": {
        "id": "c68db85fd7a47d6c3ac5d3bbe9742dac"
    },
    "response": {
        "status_code": 200,
    }
    ... (trucated)
```

## Next Step

Now you have a glance at basic usage of webhook-enterprise, you will want
to explore more features of it.

* [Basic Concepts]
* [Features]
* [Command Line Interfaces]

[Basic Concepts]: /docs/basic-concepts.md
[Installation]: /docs/installation.md
[Features]: /docs/features.md
[Command Line Interfaces]: /docs/cli.md
