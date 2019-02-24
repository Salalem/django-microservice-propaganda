# Django Propaganda

Django Propaganda is a simple pub/sub utility for [Kombu]. 

Define your broker's URL in your django settings:

```
PROPAGANDA_BROKER_URL = "amqp://guest:guest@localhost:5672//"
```

Give Propaganda a topic exchange name and it provides a nice interface for pub/sub. 

## Installation

```
pip install django-microservice-propaganda
```

## Creating Propaganda Object

```python
from django_propaganda import Propaganda

propaganda = Propaganda("exchange_topic_name")
```

## Publishing

Publish method takes routing key and payload parameters:

```python
propaganda.publish('my.routing.key', {'hello': 'world'})
```

You can omit the payload which defaults to an empty dictionary.

```python
propaganda.publish('my.routing.key')
```

`publish` method by default blocks the execution until message is sent. You can add `block=False` parameter to make 
publish return immediately, and message will be sent in a background thread.

```python
# this will return after creating a thread that will send the message
propaganda.publish('my.routing.key', {'hello': 'world'}, block=False)  
```

#### Make Persistent Additions to Payloads

Propaganda object has a dictionary property named `payload`. Content of `payload` will be added
to the payload given to the publish method.

```python
propaganda.payload['sender'] = 'Yigit Ozen'

# this will publish {'sender': 'Yigit Ozen', 'hello': 'world'}
propaganda.publish('my.routing.key', {'hello': 'world'})

# sender will still be added if you omit the payload parameter. 
# this will publish {'sender': 'Yigit Ozen'}
propaganda.publish({'my.routing.key')

# use del to stop adding a key-value pair to the payloads
del propaganda.payload['sender']
```

## Subscribing

Propaganda's `subscribe` method creates and returns a Subscription object. Subscription creates a 
queue and starts consuming it on a separate thread. 

`on` method registers a callback for a certain routing key.
 
`all` method registers a callback for all routing keys.

The signature of the callbacks must take two arguments: (body, message), which is the decoded message body and Kombu Message instance.

`wait` method blocks until a message arrives with the given routing key.

`wait_any` method blocks until any message arrives.

```python
def on_rabbit(body, message):
    print('Look, a rabbit!')
    print(body)
    
sub = propaganda.subscribe('animals.#')

# from now on when a animals.rabbit message arrives, on_rabbit will be called
sub.on('animals.rabbit', on_rabbit)

# when any message matching animals.# arrives, on_animal will be called
sub.all(on_animal)

# this will block the program until an animals.rabbit message arrives
sub.wait('animals.rabbit')

# wait has the same semantic with Python's multithreading wait. so you can pass a timeout.
# return value will be true if a message arrives before the timeout, false otherwise
arrived = sub.wait('animals.rabbit', timeout=10)

# blocks until any message arrives
sub.wait_any()
```

`on` and `all` methods return the Subscription object, so what you can chain the calls.

```python
sub = propaganda.subscribe('animals.#')
            .on('animals.rabbit', on_rabbit)
            .on('animals.turtle', on_turtle)
            .all(on_animal)
```

#### Exception Handling

You can pass an exception handler to `on` and `all` methods. Propaganda will call your handler with the exception 
when an exception is raised from on of your callbacks instead of raising it.  
For example you can use this to register loggers.

```python
def log_mq_exception(exception):
    logger.error('Exception occurred when handling MQ message: {0}'.format(exception))
    
sub = propaganda.subscribe('animals.#')

sub.on('animals.rabbit', on_rabbit, on_exception=log_mq_exception)
sub.all(on_animal, on_exception=log_mq_exception) 
```

You can also pass an exception handler to `sub` method in the same way. 
It applies to all callbacks under that subscription. 
It is called **after** the handler given to `on` or `all` method.

The following code works exactly like the one above.

```python
def log_mq_exception(exception):
    logger.error('Exception occurred when handling MQ message: {0}'.format(exception))
    
sub = propaganda.subscribe('animals.#', on_exception=log_mq_exception)

sub.on('animals.rabbit', on_rabbit)
sub.all(on_animal) 
```

## Stop and Restart Consuming

Subscription creates a queue and starts a thread that consumes it upon creation.
You can destroy the queue and the thread without destroying the Subscription, and you can recreate them later.

```python
# starts listening for the events
sub = propaganda.subscribe('animals.#')
            .on('animals.rabbit', on_rabbit)
            .on('animals.turtle', on_turtle)

# you don't want to receive messages for a while
sub.stop()

# the registered callbacks are still there.
# you can recreate the queue and start receiving messages again.
sub.start()
```

If you don't keep the subscription in a variable, you can use `unsubscribe` method of Propaganda object by binding key 
to stop consuming.

```python
propaganda.unsubscribe('animals.#')
```

## Prefixing Routing Keys

You can pass a key prefix when initializing Propaganda object, which will be automatically prefixed all routing keys 
when subscribing and publishing. If you use Propaganda object to pub/sub in a specific namespace, this will save you 
from adding the prefix manually for all pub/sub calls.

```python
propaganda = Propaganda(connection, exchange, key_prefix='my.namespace.')

# this will actually subscribe to 'my.namespace.animals.#'
sub = propaganda.subscribe('animals.#')

# on_rabbit will be called for messages with the key 'my.namespace.animals.rabbit'
sub.on('animals.rabbit', on_rabbit)

# this will publish the message with the key 'my.namespace.my.routing.key'
propaganda.publish({'my.routing.key', 'hello': 'world'})
```


[Kombu]: https://github.com/celery/kombu
