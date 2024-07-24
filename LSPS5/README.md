# LSPS5 Webhook Registration

| Name    | `webhook_registration` |
| ------- | ---------------------- |
| Version | 1                      |
| Status  | For Implementation     |

## Motivation

In typical mobile environments, a program that is not currently
being focused on by the user will be suspended, with all its
TCP connections dropped.

Typically, such a program would not be able to get either CPU
time or any ability to receive information from network.

If an LSPS client is implemented as a program on such a mobile
environment, then when it is suspended, the LSP cannot contact
the client to perform any operations, such as to forward a
payment to the client.

Generally, mobile environments will have the mobile OS developers
run a server or network of servers where an application developer
can deliver "push notifications".
When an application developer server contacts the mobile OS
developer push notification server, the application developer
server signs and sends a message that is to be displayed to the
user of the mobile unit, which the mobile OS server will validate
before causing the message to be displayed to the specified user.
If the user then taps on this "push notification" on their screen,
the application is awoken and it can then re-establish its TCP
connections.

There may be multiple implementations of mobile LSPS-supporting
clients, each of which run their own application developer server
and have their own signing keys for these push notifications.

This specification provides a way for a mobile client to
register some specific webhook by which the LSP can signal a
push notification to the application developer server, which
will in turn convert the push notification to one that it itself
signs and can send to the mobile OS developer server.

### Actors

The 'client' is a specific instance of the client application
running on some target device, usually a mobile device.

The 'LSP' is a published node that may need to awaken some
client instance in order to handle some event.

The 'notification delivery service' is the server that a
client application developer maintains, which is contacted by
the LSP via an HTTPS `POST` request, in order for the LSP
to awaken some client instance running on a target device.

The 'mobile OS server' is the entity that actually allows
push notifications to be sent to actual mobile devices.

Depending on architecture, the 'notification delivery service'
and the 'mobile OS server' may not exist.
For instance, a client running on a desktop environment might
be able to periodically wake up and pool a 'notification
delivery service' without help from any kind of 'mobile OS
server'.
A client running on an environment where some DNS-resolvable
name points to it may serve as the 'notification delivery
service' to itself.

## Protocol

### Webhook Registration API

The client can register a webhook under a specific name, list the
names of registered webhooks, and remove named webhooks.

* `lsps5.set_webhook` - Accepts an `app_name` string and `webhook`
  URL, which adds a new webhook under that name (or replaces the
  existing webhook under that name).
* `lsps5.list_webhooks` - Returns a list of `app_name` strings
  that have currently-registered webhooks.
* `lsps5.remove_webhook` - Removes the webhook with a given
  `app_name`.

If the LSP needs some action from the client, but the client is
not currently connected to the LSP, the LSP contacts all
registered webhooks.

The LSP supports some maximum number of webhooks.

> **Rationale** Each webhook not only represents some information
> that the LSP must store in persistent storage, but whenever the
> LSP needs to wake up the client, the LSP needs to contact
> **all** webhooks, thus each webhook is also a potential increase
> in network bandwidth use.

#### `lsps5.set_webhook`

The client uses the `lsps5.set_webhook` call to specify the URI
that the LSP should contact in order to send a push notification
to the client user.
The `lsps5.set_webhook` API takes the parameters:

```JSON
{
    "app_name": "My LSPS-Compliant Lightning Client",
    "webhook": "https://www.example.org/push?l=1234567890abcdefghijklmnopqrstuv&c=best"
}
```

`app_name` is a required string, containing a human-readable
UTF-8 string that gives a name to the webhook.
The `app_name` can be up to 64 bytes in UTF-8 format, with `\`
escape codes counting as multiple bytes (e.g. `\a` is two bytes,
`\000` is 4 bytes), but not counting the `"` double quote
delimiters.

> **Rationale** Backslash escapes may represent characters that
> are problematic in some contexts, such as `\000` when using C
> string libraries, and implementations may store the string
> with the backslash escapes as-is.
>
> The use of backslash escapes is dubious in `app_name`, but is
> allowed since JSON allows it.
>
> The `app_name` is specified to be human-readable, to allow
> the creation of a user interface that can manage multiple
> webhooks.

`webhook` is a required string, containing the URL of the
webhook that the LSP can use to push a notification to the
client.
The `webhook` can be up to 1024 ASCII characters, not
counting the `"` double quote delimiters.

The `webhook` MUST be a URL as defined in [RFC 1738][].
If the client is configured to use an IRI (Internationalized
Resource Identifier), the client MUST convert the IRI to a
URI first, as specified in [RFC 3987 Mapping of IRIs to URIs][].

[RFC 1738]: https://datatracker.ietf.org/doc/html/rfc1738
[RFC 3987 Mapping of IRIs to URIs]: https://datatracker.ietf.org/doc/html/rfc3987#section-3.1

The `webhook` MUST have one of the following protocols:

* `https`

The client:

* MUST use a HTTPS-protocol URI to the notification delivery
  service.
* SHOULD encode the following data in any part of the URI
  (hostname, path, or `GET` parameters):
  * The LSP node ID.
  * Some authorization to allow the given LSP to wake the
    client, ideally a proof that the client has a channel
    or other relationship with the LSP.
  * Some way to identify the target device the client is
    running on.
  * This encoding MAY be done by registering all the information
    with the notification delivery service and then getting a
    token in return, and only putting the token in the URI.
* MAY encode any additional information in the URI.

> **Rationale** By placing most data in the URI, the LSP
> interface is simplified and the client developer has the
> flexibility to add any additional information they want
> the LSP to simply pass through from the client instance
> to the notification delivery service.

If a webhook with the given `app_name` is already registered,
then this call simply replaces the existing webhook and
succeeds.
Otherwise, the LSP MUST check if the number of registered
webhooks is already at the limit it imposes.
If it is still below the limit, the LSP inserts the new
webhook under the given `app_name` and succeeds.
(i.e. this is an "insert-or-replace" operation with a maximum
number of inserted webhooks)

The LSP:

* MUST persist the given webhook and associate it with the client
  node ID and given `app_name` before responding to this call.
* MUST use **all** `webhook`s that were registered by successful
  `lsps5.set_webhook` calls, ***except*** for the
  `lsps5.webhook_registered` notification.
* MUST remember all webhooks:
  * If there are currently no channels with this client, for at
    least 7 days.
    * If a new channel with the client is opened, then as below.
  * If there is at least one open channel with this client,
    indefinitely remember them, until for at least 7 days after
    the last channel with the client is closed.

On success, `lsps5.set_webhook` returns an object like the
following:

```JSON
{
  "num_webhooks": 2,
  "max_webhooks": 4,
  "no_change": false
}
```

`num_webhooks` is the number of webhooks already registered,
including this one if it added a new webhook.
`max_webhooks` is the maximum number of webhooks the LSP allows
per client.
Both are unsigned JSON integers.

`no_change` is a Boolean.
If `true`, then the client has used `lsps5.set_webhook` before,
with the exact same `app_name` and `webhook` as the LSP has in
its storage.
If `false`, then either the `app_name` is new, or the
corresponding `webhook` was changed.

If `no_change` is `false`, then the LSP will send the
`lsps5.webhook_registered` notification to the `webhook`
specified in this call, and only to that webhook.
Only the newly-registered webhook is contacted for this
notification (unlike other events, which contact all registered
webhooks).
The LSP MAY send this notification before or after responding to
the `lsps5.set_webhook` call.
The LSP MUST send this notification to this webhook before sending
any other notifications to this webhook.

The `lsps5.set_webhook` call has the following errors defined
(error `code` number in parenthesis):

* `too_long` (500) - Either `app_name` or `webhook` or both are
  too long.
* `url_parse_error` (501) - The `webhook` failed to parse as a
  valid URL as per RFC 1738.
* `unsupported_protocol` (502) - The `webhook` is not a protocol
  that the LSP supports.
  * Future revisions of this LSPS MAY allow other protocols than
    HTTPS.
    However, an LSP that does not support that protocol is
    allowed to return this error, and the client MUST either
    set up with a protocol that the LSP supports, or not have
    webhook notifications.
    An LSP MUST support HTTPS at the minimum.
* `too_many_webhooks` (503) - The client already has the maximum
  number of webhooks that the LSP allows.
  The `data` object of the `error` response contains the field
  `max_webhooks`, a JSON unsigned integer indicating the
  maximum number of webhooks supported by the LSP.

#### `lsps5.list_webhooks`

The client can use `lsps5.list_webhooks` to learn all `app_name`s
that have webhooks registered for the client.
It takes no parameters `{}`.

`lsps5.list_webhooks` returns an object like the following:

```JSON
{
  "app_names": ["My LSPS-Compliant Lightning Wallet", "Another Wallet With The Same Signing Device"],
  "max_webhooks": 42
}
```

There are no errors defined for `lsps5.list_webhooks`.

#### `lsps5.remove_webhook`

The client can outright remove an existing registered webhook
via the `lsps5.remove_webhook` call, which takes the parameters:

```JSON
{
  "app_name": "Another Wallet With The Same Signig Device"
}
```

The following error is defined for `remove_webhook` (error
`code` in parantheses):

* `app_name_not_found` (1010) - The specified `app_name` was
  not found.

### Webhook Call

The LSP SHOULD contact *one* registered webhook URI, if:

* The client has just successfully registered that webhook
  (i.e. successful call of `lsps5.set_webhook`) AND the
  `app_name` was added, or the `app_name` was not added
  but the `webhook` is changed.

The LSP SHOULD contact all registered webhook URIs, if:

* The client has registered at least one via `lsps5.set_webhook`.
* *and* the client currently does not have a BOLT8 tunnel
  with the LSP (i.e. it is currently not connected to the
  LSP).
* *and* one of the following conditions is true:
  * The LSP learns of an incoming payment to the client
    (whether an incoming HTLC, or other future mechanism
    for incoming payments).
    * For HTLCs (or in the future, PTLCs), the LSP SHOULD hold
      onto the incoming H/PTLC for a reasonable time that
      allows the client to awaken, but short enough that the
      rest of the published network is not unduly impacted.
  * An HTLC or other time-bound contract, in either direction,
    in a channel with the client, is about to time out.
  * The LSP wants to take back some of its liquidity towards
    the client or wants to refuse serving the client.
  * The LSP has received one or more [BOLT Onion Message][]s
    for the client.
  * An event or condition specified in some other LSPS has
    occurred or become true.

The LSP generates a [JSON-RPC 2.0 Notification Object][] for
the webhook notification it wants to make.

[JSON-RPC 2.0 Notification Object]: https://www.jsonrpc.org/specification#notification

```JSON
{
  "jsonrpc": "2.0",
  "method": "lsps5.webhook_registered",
  "params": { }
}
```

`jsonrpc` is a required string field, which MUST always be
the JSON string `"2.0"`.

`method` is the required name of the notification, and MUST
be one of the webhook notification methods listed later in
this specification.
The `method` specifies exactly what the LSP wants to notify
to the client as a push notification.

`params` is a required by-name parameter object for the
notification method.
The parameters may be empty, if the method allows the
parameters to be empty.

The LSP creates a timestamp of when the notification is
created, then signs both the timestamp and the above body.

The signature is by the LSP using its node ID for signing
as described in [<LSPS0 ln_signature>][].

[<LSPS0 ln_signature>]: ../LSPS0/common-schemas.md#link-lsps0ln_signature

The message to be signed is the following JSON string template:

    "LSPS5: DO NOT SIGN THIS MESSAGE MANUALLY: LSP: At ${timestamp} I notify ${body}"

Where `${timestamp}` is the timestamp, in ISO8601
date format `YYYY-MM-DDThh:mm:ss.uuuZ` [<LSPS0 datetime>][],
and `${body}` is the exact and complete JSON serialization of
the above webhook notification object.

[<LSPS0 datetime>]: ../LSPS0/common-schemas.md#link-lsps0datetime

For example, if the timestamp is `2023-05-04T10:52:58.395Z`
and the webhook notification object is the exact ASCII sequence
`{"jsonrpc":"2.0","method":"lsps5.goodbye","params":{}}`,
then the JSON string to be signed is:

    "LSPS5: DO NOT SIGN THIS MESSAGE MANUALLY: LSP: At 2023-05-04T10:52:58.395Z I notify {\"jsonrpc\":\"2.0\",\"method\":\"lsps5.goodbye\",\"params\":{}}"

The message to be signed is the contained string in UTF-8
format, without a trailing `NUL` character, and with the
escape characters processed as per standard JSON string
escaping rules (e.g. `\"` would be the single byte 0x22,
not the two bytes 0x5C 0x22).
For the above example, the message to be signed would have the
following hex dump:

```
00000000: 4c53 5053 353a 2044 4f20 4e4f 5420 5349  LSPS5: DO NOT SI
00000010: 474e 2054 4849 5320 4d45 5353 4147 4520  GN THIS MESSAGE
00000020: 4d41 4e55 414c 4c59 3a20 4c53 503a 2041  MANUALLY: LSP: A
00000030: 7420 3230 3233 2d30 352d 3034 5431 303a  t 2023-05-04T10:
00000040: 3532 3a35 382e 3339 355a 2049 206e 6f74  52:58.395Z I not
00000050: 6966 7920 7b22 6a73 6f6e 7270 6322 3a22  ify {"jsonrpc":"
00000060: 322e 3022 2c22 6d65 7468 6f64 223a 226c  2.0","method":"l
00000070: 7370 7335 2e67 6f6f 6462 7965 222c 2270  sps5.goodbye","p
00000080: 6172 616d 7322 3a7b 7d7d                 arams":{}}
```

The resulting signature is a zbase32-formatted string.
See [<LSPS0 ln_signature>][] for more information.

The timestamp, signature, and webhook notification object are sent
to the notification delivery service based on the protocol of the
webhook:

* For HTTPS webhooks, the LSP makes a `POST` request, with
  the webhook notification object as the exact `POST` request
  body, and with additional HTTP headers below:
  * `x-lsps5-timestamp` - the LSP-side timestamp, in ISO8601
     date format `YYYY-MM-DDThh:mm:ss.uuuZ`, for example
    `2023-05-04T10:14:23.853Z`.
  * `x-lsps5-signature` - the LSP-generated signature, as a
    zbase32 string.

The notification delivery service, on being contacted via a
registered webhook:

* For HTTPS webhooks:
  * MUST check that the webhook was contacted via a
    `POST` HTTPS request.
  * MUST validate that the URI (hostname, path, and/or
    `GET` parameters) contains:
    * The LSP node ID.
    * A valid authorization for the LSP to wake the client.
    * A way to identify the target device(s) the client is
      running on.
    * Or some token that it issued on registration of the
      above information.
  * MUST check that the `x-api-timestamp` and
    `x-api-signature` HTTP headers exist and are the
    expected formats.
  * MUST validate that the `POST` request body is a valid
    [JSON-RPC 2.0 Notification Object][], which is the
    webhook notification object.

The notification delivery service, once it has received
the timestamp, signature, and webhook notification object:

* MUST check that the timestamp is within 10 minutes
  (before or after) of its local time.
* MUST remember the signature for at least 20 minutes,
  and check that the current signature is not equal to
  some remembered signature from a previous webhook
  call.
* MUST validate the signature against the message
  template above, with the given timestamp and webhook
  notification object.
* MAY filter out particular notification methods by
  its own policy.

> **Rationale** The timestamp and signature checks are
> intended to avoid replay attacks, where some third
> party is able to copy the JSON object sent by the LSP
> and replays it in order to frame the LSP for spamming
> the client.

The response to the webhook is unspecified.

* For HTTPS webhooks (i.e. `POST`):
  * The LSP MUST consider a `200 OK` response a
    success, and MUST ignore the rest of the response,
    including HTTP headers and the response body.
  * If the response is not `200 OK`, the LSP SHOULD treat this as
    an unusual event, and MUST otherwise ignore the response.
  * The LSP SHOULD NOT follow any redirection responses.

> **Rationale** As a notification, the user of the
> client may ignore it, or disable notifications from
> the client application entirely; thus, delivery of
> the notification is never assured and it is pointless
> to feed back anything to the LSP about whether the
> notification was delivered to the notification
> delivery service, the mobile OS server, or the client
> mobile device.

The notification delivery service MAY reject the push
notification via its own policies, such as if the LSP has been
making too many push notifications in a short period, or if the
client device is not contactable in any way.

If the notification delivery service instead accepts the push
notification, it MUST construct a suitable message to display to
the user, and send that message, together with any required
authentication and authorization, to the mobile OS server, for
display as a push notification of the client.

The client SHOULD process what is needed to
react to the notification (e.g. connect to the LSP and
accept the payment for a `lsps5.payment_incoming`
notification) if the mobile environment allows it.

#### Webhook Notification Methods

The LSP SHOULD use one of the methods listed in this
section for the webhook notification object.

A different LSPS, or a non-standard extension to the common
LSPS specifications, MAY specify additional webhook notification
methods, as well as the conditions or events that would
cause the LSP to emit them.
The LSP MAY use such notification methods if it supports
that LSPS or non-standard extension.
Such non-LSPS5 notification methods MUST NOT use the
`lsps5.` prefix, which is reserved for LSPS5, and SHOULD
use their own prefix.

The LSP MUST NOT use any of these methods on the LSPS0
interface; these notification methods are only for the
webhook interface.

Parameters are required unless otherwise stated.

* `lsps5.webhook_registered` - The client has just
  recently successfully called the `lsps5.set_webhook`
  API.
  Only the newly-(re)registered webhook is notified.
  No parameters `{}`.
* `lsps5.payment_incoming` - The client has one or
  more payments pending to be received.
  No parameters `{}`.
* `lsps5.expiry_soon` - There is an HTLC or
  other time-bound contract, in either direction, on
  one of the channels between the client and the LSP,
  and it is within 24 blocks of being timed out, and
  the timeout would cause a channel closure.
  Parameters:
  * `timeout` - The block height at which the LSP would
    be forced to close the channel in order to enforce
    the HTLC or other time-bound contract.
* `lsps5.liquidity_management_request` - The LSP wants
  to take back some of the liquidity it has towards the
  client, for example by closing one or more of the
  channels it has with the client, or by splicing out.
  No parameters `{}`.
* `lsps5.onion_message_incoming` - The client has one
  or more [BOLT Onion Messages][Bolt Onion Message]
  pending to be received.
  No parameters `{}`.

[BOLT Onion Message]: https://github.com/lightning/bolts/blob/bccab9afc269a50a579850e71b06e58ef16c4e92/04-onion-routing.md#onion-messages

Future revisions of this LSPS MAY define new notification
methods.
Notification delivery services MUST ignore any notification
method it does not recognize.

Future revisions of this LSPS MAY define new parameters for
existing notification methods.
Notification delivery services MUST ignore any parameters it does
not recognize.

### Implementation Quality Guidelines

> **Non-normative** While these guidelines are not technically
> required in order to comply with this standard,
> production-quality, non-beta software should really follow
> these guidelines.

#### Mobile Client: Privacy On Mobile OS Notification

[There are claims that governments have used push notifications
to spy on users][push-notif-surveill].

[push-notif-surveill]: https://web.archive.org/web/20231211010610/https://www.reuters.com/technology/cybersecurity/governments-spying-apple-google-users-through-push-notifications-us-senator-2023-12-06/

To avoid this, a quality implementation of an LSPS5 notification
delivery service that has to talk to a mobile OS server must
encrypt as much data as possible when constructing the notification
to be sent to the mobile OS server, to be delivered to a client
application on a mobile device.

In particular, it is expected that all notifications from the
notification delivery service would include an LSP node ID.
If a client is using multiple LSP nodes, then it would need
to know which one to contact once it gets the notification.

> **Rationale** We *could* use an end-to-end encryption from the
> LSP to the client, so that even the notification delivery
> service is unaware of the content of the notification.
>
> However, if a third party is able to determine the URL naming
> schema used by a notification delivery service, they would be
> able to spam synthetic URLs by simply POSTing to them.
> If notifications were encrypted end-to-end, then a spammer
> would be able to force mobile devices to wake up (and drain
> battery) to decrypt the notifications, because only the client
> would be able to decrypt it anyway.
> The current LSPS5 design requires that the LSP reveal its
> node ID and signature to the notification delivery service,
> to prevent this form of spam.
>
> As-is, the highest-entropy data that needs to be sent to the
> client is actually the LSP node ID that is waking it up.
> And this LSP node ID has to be known by the notification
> delivery server in order to filter out the spamming attack
> above.

The encryption must use well-designed cryptosystems, such as,
but not limited to, encrypting the push notification with
AEAD using SECP256K1 to the client node ID.

#### Client: Webhook URL Schema

There is no need for a special mechanism by which a client
can "pass through" information about the LSP or its channel(s)
with the LSP via the notification body itself.

A client can encode all information that it wants the LSP to
pass through verbatim in the URL itself, such as by `GET`
parameters in an HTTPS webhook, or part of the path, or the
DNS name.

Thus, information like the client identifier or the LSP
Lightning Network node identifier can be encoded by the client
in the URL, and the notification delivery service can then
extract it from the URL that is requested by the LSP.

However, the URL naming schema is likely to be discoverable.
For instance, if the client is open-source, then the URL
naming scheme would also be revealed.

Spammers may then attack a client by posing as LSPs, and
then signing the notifications using uniquely-generated
keypairs.

To protect against this, both the client and its notification
delivery service need to agree on some validation that
the URL could only have been constructed by the client.

For example, the URL might encode not only the client
node ID and the LSP node ID, but also a signature of the
client, signing off on the LSP node ID (as well as any
other parameters embedded in the URL).
Then, a spammer would need the client private key in order
to spoof an arbitrary LSP node ID and spam the client.

Alternatively, the notification delivery service might
assign a unique large number in a vast 256-bit space to
the client.
The spammer would then need to scan a vast space just to
hit on some client.

#### LSP: Rate-limiting Notifications By `method`

An LSP implementation must avoid sending multiple LSPS5
notifications of the same `method` (other than
`lsps5.webhook_registered`) close in time, as long as the client
has not connected to it.

For example, if a payment arrives and the LSP thus
wants to send `lsps5.payment_incoming` notification to a
registered webhook, and another payment arrives before the
client comes online, the LSP must not send another
`lsps5.payment_incoming` notification.

If the client does not come online after some time that a
particular `method` was sent via a webhook, then the LSP
may raise it again.
This timeout must be measurable in hours or days.

The timeout should be reset once the client comes online
and then goes offline.
