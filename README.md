# read-through-cache

  a simple read-through on-disk cache.

## Design goals

  * Be agnostic to what is being cached. As long as it arrives as a readable bytestream, it's cacheable.
  * Combine as many requests for the same thing as possible.
    Until the first byte flows out of the ReadStream, it combines every request into one.
  * Be optimistic.
    Don't check the cache and *then* get things from it. Try to get things from it, and if we hit an `ENOENT`, fall back.
  * Remain correct in the face of concurrency.
    We go to great lengths to ensure processes wait for each other and share work.
    Even across processes, requests are combined, safely.
    Only process-local failure is considered a hard failure.
  * Use as few file descriptors as possible.
    File descriptors are often scarce, and many operations can be run on an existing fd.

## API
### Cache(path)
### Cache({ path, timeout })

  Returns an instance of the cache, storing its stuff at the given path.

  If a timeout is given (in milliseconds), the staleness check kicks in after 100ms of waiting for a temporary file.
  A working process resets the timeout on every write and every half timeout period.
  If the timeout passes, we remove the temporary file and try again.

#### cache.createReadStream(key, …args)

  Returns a readable stream for the item in the cache with given key. If the item is not found in the cache, it defers to `cache._createReadStream(key, …args)`.
  This may emit an error on the stream, but never a single byte of wrong content.

#### cache._createReadStream(key, …args)

  Should return a readable stream that hopefully has contents matching the key. All the arguments to `cache.createReadStream(key, …args)` are simply passed on.

#### cache._storePath(key)
#### cache._tmpPath(key)

  Should return a path in the given place.

  Default implementations are provided:

```javascript
Cache.prototype._storePath = function(key) { return Path.join(this.path, 'store', key) }
Cache.prototype._tmpPath = function(key) { return Path.join(this.path, 'tmp', key) }
```

## Example

```javascript
'use strict';
module.exports = MyCache
var Cache = require('cache')
  , crypto = require('crypto')
  , http = require('http')
  , through = require('through')

function MyCache(opts) { Cache.call(this, opts) }

MyCache.prototype._storePath = function(key) { return Path.join(this.path, 'store', key.slice(0, 2), key) }
MyCache.prototype._tmpPath = function(key) { return Path.join(this.path, 'tmp', key.slice(0, 2), key) }

MyCache.prototype._createReadStream = function(key, url) {
  var stream = through()
  http.get(url)
    .on('response', function(res) {
      if (res.statusCode === 200) return res.pipe(stream)
      var err = new Error('HTTP ' + res.statusCode)
      err.statusCode = res.statusCode
      stream.emit('error', err)
    })
    .on('error', function(err) { stream.emit('error', err) })
  return stream
}
```

