Angaza Backend Exercise
=======================

This environment encapsulates a small and self-contained exercise in backend
engineering. It is not intended to be particularly complicated, confusing, or
time-consuming. There are no "gotchas". It does, however, require you to work
with a small but realistic backend system, with TODOs, warts, limited
documentation, and missing functionality.

For avoidance of all doubt, we want to be clear that we took an *existing*
feature in our production backend and stripped it out for this exercise. We
are *not* attempting to get "free work" from you!

Backstory
---------

ACME PayCo operates a backend system that sends SMS messages in response to
payment notifications. It performs this simple task well, but a new business
need has arisen: the system needs to track the *cost* of the SMS messages that
it sends, so that these costs can be tracked and reported, especially because
the cost of a message can change depending on the destination. That cost can
also change over time!

The system uses Twilio to send SMS messages, and Twilio does in fact report the
cost of a message sent through its system. Right now, however, that information
is neither retrieved nor stored. That's where you come in.

Your Mission
------------

You should implement this cost-tracking feature. After you are done:

- The cost of each SMS should be stored in the database.
- The cost of each SMS should be reflected in the JSON representation returned
  by the HTTP API.

After implementing this feature, it should be possible to see the cost of a
given SMS in the response returned by a request along the lines of:

```
$ http -p b get http://payg.localhost/api/sms_messages/SM4

{
    "_links": {
        "self": {
            "href": "https://payg.localhost/api/sms_messages/SM4"
        }, 
        "type": {
            "href": "/types/sms_message"
        }
    }, 
    "body": "Your payment of USD 42 was received!", 
    "cost": {
        "_type": "decimal", 
        "value": "0.10"
    }, 
    "cost_currency": "USD", 
    "created_when": {
        "_type": "iso8601", 
        "value": "2015-05-20T21:58:30.824686"
    }, 
    "error": null, 
    "id": 4, 
    "recipient": "+15551234567", 
    "sent_when": {
        "_type": "iso8601", 
        "value": "2015-05-20T21:58:30.925150"
    }, 
    "state": "FINAL_DELIVERED"
}
```

Various Details
---------------

### Directories and Database(s)

All the code you'll need to modify is in this directory. You can use a text
editor on your host to edit any source file in the synched directory, and use
SSH access to the guest to run the modified project. Most of the instructions
in this document assume that you are executing commands from the
`~/Projects/Angaza` directory while SSHed into the guest.

You have passwordless `sudo` access in the guest, although you typically will
not need to use it.

We have created a PostgreSQL database named `za_sandbox` in the guest. As
configured, this database will be used by the backend code when you follow the
outlined steps below. You may use the `za-dbfill` tool (sample invocation
below) to re-create an empty database with tables and columns defined by our
object models (e.g., a `User` object is persisted in the `users` table). For
the purposes of this exercise, you may assume that the database schema will
always match your object models (i.e., you don't need to write any schema
migrations).

### Twilio

To send actual SMS messages, you should create a Twilio trial account. Doing so
is completely free; no credit card is required. Visit twilio.com, choose "Sign
Up", and follow the instructions. Then head to [Programmable SMS
Dashboard](https://www.twilio.com/console/sms/dashboard). Click "Show API
Credentials" and use these values when starting your celery worker (as outlined
below).

Your number can be obtained on the [Phone
Number](https://www.twilio.com/console/phone-numbers/incoming) page. Ensure
that your phone number is registered with Twilio so that you can send SMSes to
yourself to test your code. You can register your number using their "Verified
Caller IDs" page.

### Running the Backend

This section walks you through running the code that you will then modify to
incorporate the feature above. All the following steps assume that you are
operating from within the Vagrant environment, inside the `~/Projects/Angaza`
directory. For this section, you will also likely want to open multiple
terminals, each running `vagrant ssh`.

Start development by verifying that the existing test suite passes:

```
nosetests za
```

You can now initialize the database and fire up the actual backend, starting
with the HTTP API:

```
za-dbfill -wipe -url postgres:///za_sandbox dev
za-serve-gunicorn biz_api:/api
```

In a separate terminal, start a Celery background worker:

```
export TWILIO_ACCOUNT_SID=YYY
export TWILIO_AUTH_TOKEN=ZZZ
export TWILIO_NUMBER=NNN
celery --app=za worker --loglevel=INFO
```

With these two backend processes running, go ahead and create some user by
making a request to the HTTP API:

```
http post http://payg.localhost/api/users \
    role=admin \
    username=someuser \
    email=someuser \
    first_name=First \
    last_name=Last \
    primary_phone=+15551234567 \
    password=swordfish \
    login_access_enabled=true
```

We are using the nice [httpie](http://httpie.org) tool to make the HTTP request
here, but cURL or a similar tool should work fine as well.

Now, to do something a bit more interesting, you can initiate a new payment
notification:

```
http --auth op=someuser:raw=swordfish post http://payg.localhost/api/payments \
    phone=+15551234567 \
    amount=50
```

Note that HTTP Basic authentication is used, based on the credentials
established in the preceding request.

If you used your actual phone number above, that action should have kicked off
a real SMS message to your phone. You can execute

```
http -p b get http://payg.localhost/api/sms_messages
```

to see the backend's representation of the new message.

Finishing Up
------------

After your feature is neatly implemented and committed to the local git
repository, send us the finished product. Clone the repo and tar it up:

```
git clone --bare backend backend.YOURNAME
tar czvf backend.YOURNAME.tar.gz backend.YOURNAME/
```

After doing so, email us that tarball.

Further Reading
---------------

Some pointers to documentation that may be helpful:

- http://www.sqlalchemy.org/
- http://flask.pocoo.org/docs/0.10/quickstart/
- http://nose.readthedocs.org/en/latest/usage.html
- http://www.celeryproject.org/
- http://tools.ietf.org/html/draft-kelly-json-hal-06
