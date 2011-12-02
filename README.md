
## Overview
Redis Queue is a simple (under 200sloc) but complete queueing system built on top of Redis. It can both schedule and prioritise queued items.

Queues are created when values are added to it. All input is encoded into JSON when stored and decoded when dequeued.

## Dependencies

Redis version 2.4.2 or higher
Ruby version 1.9.2

## Install

    $ bundle install --path vendor

## Spec
    
    $ bundle exec guard

## Usage

### Connect

For the in-memory backend that only stores keys within a process space,

    redis_queue = MobME::Infrastructure::RedisQueue.queue(:memory, options = {})

For the redis backend,

    redis_queue = MobME::Infrastructure::RedisQueue.queue(:redis, options = {})
    
For the zeromq backend,
  
    redis_queue = MobME::Infrastructure::RedisQueue.queue(:zeromq, options = {})
    
& for the AMQP backend using bunny,

    redis_queue = MobME::Infrastructure::RedisQueue.queue(:amqp, options = {})

    
All three have exactly the same public interfaces.

### Add an item

    redis_queue.add("publish", {:jobid => 23, :url => 'http://example.com/' })
    
Items can also have arbitrary metadata. They are stored alongside items and returned on a dequeue. 

    redis_queue.add("publish", {:jobid => 23, :url => 'http://example.com/' }, {'importance' => low})

Certain metadata have special meaning. If you set a dequeue-timestamp to a Time object, the item will only be dequeued *after* that time. Note that it won't be dequeued exactly *at* the time, but at any time afterwards.

    # only dequeue 5s after queueing
    redis_queue.add("publish", {:jobid => 23, :url => 'http://example.com/' }, {'dequeue-timestamp' => Time.now + 5 })

Another special metadata keyword is priority.

    # priority is an integer from 1 to 100. Higher priority items are dequeued first.
    redis_queue.add("publish", {:jobid => 23, :url => 'http://example.com/' }, {'priority' => 5})

Items with priority set (or a higher priority) are always dequeued first.

Note that the AMQP backend doesn't support priorities or the dequeue timestamp.

### Remove an item

    # dequeue
    redis_queue.remove("publish")
    
\#remove returns an array. The first element is the Ruby object in the queue, the second is the associated metadata (always a Hash).

    => {:jobid => 23, :url => 'http://example.com/' }, {'priority' => 5}
    
\#remove also can take a block. This is the recommended way to remove an item from a queue.

    # dequeue into a block
    redis_queue.remove do |item|
      #process item
      ...
    end
    
When a block is passed, RedisQueue ensures that the item is put back in case of an error within the block.

Inside a block, you can also manually raise {MobME::Infrastructure::RedisQueueRemoveAbort} to put back the item:

    # dequeue into a block
    redis_queue.remove do |item|
      #this item will be put back
      raise MobME::Infrastructure::RedisQueue::RemoveAbort
    end
    
Note: you cannot pass in a block using the zeromq or amqp queue types.
    
### List all items in a queue

This is an expensive operation, but at times, very useful!

    redis_queue.list

This is not supported for the amqp queue type.

### List available queues

    redis_queue.list_queues

Returns an array of all queues stored in the Redis instance.

### Remove queues

This empties and removes all queues:

    redis_queue.remove_queues

To selectively remove queues:

    redis_queue.remove_queue "queue1"
    redis_queue.remove_queues "queue1", "queue2"

## Performance & Memory Usage

See detailed analysis in spec/performance.

### The Redis Backend

An indicative add performance is around 100,000 values stored in 20s: 5K/s write.

An indicative normal workflow performance is 200,000 values stored and retrieved in 1 minute: ~3K/s read-write

It's also reasonably memory efficient because it uses hashes instead of plain strings to store values. 200,000 values used 20MB (with each value 10 bytes).

### The Memory Backend

The memory backend only stores keys within the process space.

But performance is *very* good. It does 200,000 read/write in around 5s, which is ~40K/s read/write.

### The ZeroMQ Backend

The zeromq backend is currently experimental. It's meant to do these things:

* Very fast queue adds (5s for 100,000 keys)
* Consistent reads
* Eventual consistency via a Redis backend (this is currently not implemented)
* A listener based queue interface where a client can request a message rather than messages being pushed down the wire (i.e. 'subscribe' to a queue) (again, not implemented yet)

### The AMQP Backend

The amqp backend uses the excellent bunny gem to connect to RabbitMQ.

This is slightly slower than the Redis backend: 200,000 values read-write in around 1m30s (~2K/s read-write)

# {include:file:TODO.md}
