# Basic Concepts

## Webhook

Webhook is callback based on HTTP. Webhooks allow target service
subscribing certain events from source target.  When something
happens, the source service makes an HTTP request to the URI
pre-registered as webhook. For the target service, webhooks are
a way to receive message that sometimes you don't want to miss,
rather than polling nothing from source service most of times.

## Source

Source services normally adopt event sourcing methodology. They
require owners of target services configure URI as webhook in
their system in advance. Then webhooks will be triggered each time
one or more subscribed events occurs in source services
In many cases, events are sent with extra data as payloads.

## Target

Target services usually do nothing more but to implement handlers
for specific webhook URIs.

The reality is that it's never enough. We need to log messages sent
from source services for tracing back. We also need to rerun webhooks
in case of not running due to network error.  You name it.

But as long as you have Webhook-Enterprise integrated into system,
things get easier then. It's his duty to store, proxy,
retry, relay events. All you need to care is to implement your own
business logic!

## Event

In Webhook-Enterprise, the HTTP request sent by source service is
called `Event`. Event contains important information that source
service want to tell to the world. Target service subscribes normally
all events but only handle some specific events by `type` field.
