---
title: ".Net Sockets and You"
layout: post
category : tips
tagline: "Doing Sockets Correctly"
tags : [tips, csharp, sockets]
---

Writing a socket server in .Net is one of the most rewarding things you can do in .Net. Unfortunately at the same time there is a lot of pre-requisite knowledge
– and the knowledge isn't easy to find online (in fact it is very easy to examples and tutorials that are blatantly wrong).

After reading this you should know enough to write a pretty good socket server; and hopefully where to start looking when bad things happen. This post is
written progressively, meaning I may use certain bad practices in the first few sections. **Don't just go and copy and paste** – make sure you read the whole
thing.

## IOCP (I/O Completion Ports)

IOCP is probably the most important part of a socket server. If you have ever heard of the Berkely socket pattern; this is the exact opposite. Berkely relies
on a tight loop that queries each socket asking it if it has any data ready; it doesn't scale. Another often-used (and incorrect) pattern is the multi-threaded
Berkely socket pattern. Instead of a single tight loop querying each socket, the server assigns a thread-per-client. While in theory this is a good idea;
realise that spinning up 1000 tight loops is going to bring your server to a screeching halt very quickly.

So what is IOCP? Put simply; it's a way for a program to say to the kernel "Hey, I am in charge of these streams. When data arrives on ANY of them please resume
this thread I have created for you." The .Net BCL team took IOCP a step further and shield developers from having to manage that thread. Using IOCP in .Net is
as easy as implementing your server using the [Begin/End async pattern.][1] Using this pattern is quite simple:

1. Create your own `Begin*` method (as a bad example, `BeginReceive`).
2. Create a stub for the `End*` method. This method should accept a single `IAsyncResult` as a parameter.
3. In your `Begin*` method, call `socket.Begin*` passing your `End*` as a delegate parameter.
4. In your `End*` method, call `socket.End*` passing the `IAsyncResult` you got as a parameter.
5. Then call your `Begin*` method.

This sets up an async loop; there is a shorter form of this loop (designed for DRY error handling) – which looks like this:

    private Socket _socket;
    private ArraySegment<byte> _buffer;
    public void StartReceive()
    {
        ReceiveAsyncLoop(null);
    }

    // Note that this method is not guaranteed (in fact
    // unlikely) to remain on a single thread across
    // async invocations.
    private void ReceiveAsyncLoop(IAsyncResult result)
    {
        try
        {
            if (result != null)
            {
                int numberOfBytesRead = _socket.EndReceive(result);
                if(numberOfBytesRead == 0)
                {
                    OnDisconnected(null); // 'null' being the exception. The client disconnected normally in this case.
                    return;
                }

                var newSegment = new ArraySegment<byte>(_buffer.Array, _buffer.Offset, numberOfBytesRead);
                OnDataReceived(newSegment);
            }
            _socket.BeginReceive(_buffer.Array, _buffer.Offset, _buffer.Count, SocketFlags.None, ReceiveAsyncLoop, null);
        }
        catch (Exception ex)
        {
            // Socket error handling here.
        }
    }

Note that you should use the async pattern for everything: `Accept`, `Receive`, `Send`, `Connect`. As a side-note – using `BeginSend` with a null callback is
fine (fire and forget); but it will play havoc with the socket performance counters for your process. You may also want to set [UseOnlyOverlappedIO][2] to true.
I also suggest you look at [Reactive Extensions;][3] as one of the things it deals with is the async pattern.

## Use Streams

Don't use the built-in `Receive` and `Send` methods, construct a `NetworkStream` over the socket. This is important because it allows you wrap your socket in a
[SslStream][4] or [DeflateStream][5] (side note: always wrap the data in compression BEFORE encryption).

`DeflateStream` was implemented in a really strange way (the existence of the `CompressionMode` parameter). You are typically going to write a `Stream` that
uses one `DeflateStream` (in `CompressionMode.Compress`) to write and another (in `CompressionMode.Decompress`) to read. Also note that in prior to .Net 4.0
`DeflateStream` should only be used to compress English/markup as it has a hard-coded dictionary.

## The Evils of Pinning

The Microsoft CLR uses P/Invoke to provision sockets. A side-effect of P/Invoke is that any reference types (and a byte array a.k.a buffer is a reference type)
that you pass as parameters are pinned for the duration of the call. Pinning is a process whereby the garbage collector is informed that it should not move an
object around (the garbage collector moves objects around to ensure that large continuous areas of free space are available). What this means is that if you
pass a new buffer to each `Send`/`Receive` call you will wind up with a lot of pinned objects, which can lead to fragmentation. There have been stories of
processes getting so fragmented that a process using 800MB of memory and 1.2GB of free memory throwing an `OutOfMemoryException` (because it could not find a
large enough space to allocate an object). Remember that an `OutOfMemoryException` kills your .Net process without ever calling any catch blocks – not good.

The way to get around this is to pre-allocate large blocks of memory for buffers. The way to get around this is to pre-allocate large blocks of memory for
buffers. `ArraySegment` is a good way to get handles on ‘sub-arrays'. Here is an effective buffer pool to get you started:

    /// <summary>
    /// Represents a buffer pool.
    /// </summary>
    public class BufferPool
    {
        private readonly int _segmentsPerChunk;
        private readonly int _segmentSize;
        private readonly ConcurrentStack<ArraySegment<byte>> _buffers;

        /// <summary>
        /// Gets the default instance of the buffer pool.
        /// </summary>
        public static readonly BufferPool Instance = new BufferPool(
            64,
            4096, /* Page size on Windows NT */
            64
            );

        /// <summary>
        /// Gets the segment size of the buffer pool.
        /// </summary>
        public int SegmentSize
        {
            get
            {
                return _segmentSize;
            }
        }

        /// <summary>
        /// Gets the amount of segments per chunk.
        /// </summary>
        public int SegmentsPerChunk
        {
            get
            {
                return _segmentsPerChunk;
            }
        }

        /// <summary>
        /// Creates a new chunk, makes buffers available.
        /// </summary>
        private void CreateNewChunk()
        {
            var val = _segmentsPerChunk * _segmentSize;

            byte[] bytes = new byte[val];
            for (int i = 0; i < _segmentsPerChunk; i++)
            {
                ArraySegment<byte> chunk = new
                    ArraySegment<byte>(bytes, i * _segmentSize, _segmentSize);
                _buffers.Push(chunk);
            }
        }

        /// <summary>
        /// Creates a new chunk, makes buffers available.
        /// </summary>
        private void CompleteChunk(byte[] bytes)
        {
            for (int i = 1; i < _segmentsPerChunk; i++)
            {
                ArraySegment<byte> chunk = new
                    ArraySegment<byte>(bytes, i * _segmentSize, _segmentSize);
                _buffers.Push(chunk);
            }
        }

        /// <summary>
        /// Checks out a buffer from the manager.
        /// </summary>
        /// <remarks>
        /// It is the client's responsibility to return the buffer to the manger by
        /// calling <see cref="CheckIn" /> on the buffer.
        /// </remarks>
        /// <returns>A <see cref="ArraySegment{Byte}" /> that can be used as a buffer.</returns>
        public ArraySegment<byte> Checkout()
        {
            ArraySegment<byte> seg = default(ArraySegment<byte>);
            if(!_buffers.TryPop(out seg))
            {
                // Allow the caller to continue as soon as possible.
                var chunk = new byte[_segmentsPerChunk * _segmentSize];
                var action = new Action<byte[]>(CompleteChunk);
                action.BeginInvoke(chunk, x => action.EndInvoke(x));
                // We have the buffer at the start of the chunk.
                seg = new ArraySegment<byte>(chunk, 0, _segmentsPerChunk);
            }

            return seg;
        }

        /// <summary>
        /// Returns a buffer to the control of the manager.
        /// </summary>
        /// <param name="buffer">The <see cref="ArraySegment{Byte}"></see> to return to the cache.</param>
        public void CheckIn(ArraySegment<byte> buffer)
        {
            if (buffer.Array == null)
                throw new ArgumentNullException("buffer.Array");
            _buffers.Push(buffer);
        }

        /// <summary>
        /// Constructs a new <see cref="BufferPool" /> object
        /// </summary>
        /// <param name="segmentChunks">The number of chunks to create per segment</param>
        /// <param name="chunkSize">The size of a chunk in bytes</param>
        public BufferPool(int segmentChunks, int chunkSize) :
            this(segmentChunks, chunkSize, 1)
        {

        }

        /// <summary>
        /// Constructs a new <see cref="BufferPool"></see> object
        /// </summary>
        /// <param name="segmentsPerChunk">The number of segments per chunk.</param>
        /// <param name="segmentSize">The size of each segment.</param>
        /// <param name="initialChunks">The initial number of chunks to create.</param>
        public BufferPool(int segmentsPerChunk, int segmentSize, int initialChunks)
        {
            _segmentsPerChunk = segmentsPerChunk;
            _segmentSize = segmentSize;
            _buffers = new ConcurrentStack<ArraySegment<byte>>();
            for (int i = 0; i < initialChunks; i++)
            {
                CreateNewChunk();
            }
        }
    }

Note that you can also use [BufferManager][6] (which doesn't support `ArraySegment`).

Unfortunately this one is actually really hard to get 100% right in the scenario where you do use `SslStream` and `DeflateStream`. These two streams allocate
their own buffers; which means even if you do write to the `targetStream` using pooled buffers, unpooled buffers will still be sent to the underlying socket. If
you do run into this scenario you are going to land up writing quite a bit of code (subclassing `Stream`; placed directly above your `NetworkStream`) that swaps
out the buffers passed to it for pooled buffers (keeping in mind you need to do this async).

There are also two ways to manage these pooled buffers. If your server behaves in a streaming fashion (which is probably the case) you will want to experiment
with allocating a buffer to each client; instead of a buffer to each `Read`/`Write` call. Whether this will work out better or not depends on the nature of
your server.

## DOS

Denial-Of-Service (not that I didn't mention DDOS; pretty-much the only way to deal with that is horizontal scaling) is one you need to look out for. DOS is
really simple to pull off if you don't protect yourself against it. It merely comes down to asking "why is this client connected to me?", "can I handle this
much data?" and for the love of God staying away from byte arrays (except for, obviously, at the socket level). For instance, what is a client doing connected
to you if it has not authenticated itself in 30 seconds? Why is this client sending you 5GB of data? Can the data be streamed to disk instead?

## PUSH Parsing

Make sure that your server can PUSH parse. One form of DOS is to send a really big (but valid) packet. Naïve servers will build up a large buffer in memory
to attempt to store this packet before parsing it - potentially overallocating and crashing the process. Don't do this, instead parse the bytes as you get them.
For instance instead of using a `StreamReader` look at the following:

    private Encoding _encoding;
    private Decoder _decoder;
    private char[] _charData = new char[4];

    public PushTextReader(Encoding encoding)
    {
        _encoding = encoding;
        _decoder = _encoding.GetDecoder();
    }

    // A single connection requires its own decoder
    // and charData. That connection should never
    // call this method from multiple threads
    // simultaneously.
    // If you are using the ReadAsyncLoop you
    // don't need to worry about it.
    public void ReceiveData(ArraySegment<byte> data)
    {
        // The two false parameters cause the decoder
        // to accept 'partial' characters.
        var charCount = _decoder.GetCharCount(data.Array, data.Offset, data.Count, false);
        charCount = _decoder.GetChars(data.Array, data.Offset, data.Count, _charData, 0, false);
        OnCharacterData(new ArraySegment<char>(_charData, 0, charCount));
    }

If you need to build up the packet before parsing it (say it's XML and you don't want to write/test your own PUSH parser) rather write it to a temporary file.

## Hope This Helps

If you can think of anything I should add or correct please let me know, or [fork this post][7] and send a pull request.

[1]: http://msdn.microsoft.com/en-us/library/aa719595.aspx
[2]: http://msdn.microsoft.com/en-us/library/system.net.sockets.socket.useonlyoverlappedio.aspx
[3]: http://msdn.microsoft.com/en-us/devlabs/ee794896
[4]: http://msdn.microsoft.com/en-us/library/system.net.security.sslstream.aspx
[5]: http://msdn.microsoft.com/en-us/library/system.io.compression.deflatestream.aspx
[6]: http://msdn.microsoft.com/en-us/library/system.servicemodel.channels.buffermanager.aspx
[7]: https://github.com/jcdickinson/jcdickinson.github.io/blob/master/_posts/2015-06-01-Net-Sockets-and-You.md
