# torii's fork of h2 0.4.20

This is the `torii` branch: h2 0.4.20 (https://github.com/hyperium/h2) plus the
changes below, for [torii](https://github.com/Dirbaio/torii)'s data plane. Every change is marked
`torii fork` in the source. The goal: an idle HTTP/2 connection holds no I/O
buffers, as in nginx.

| Change | Where | Upstream behaviour |
| --- | --- | --- |
| Frames are read by `FrameReader`, whose buffer is allocated when a read starts and dropped when a read would block with nothing buffered | `src/codec/framed_read.rs` | tokio-util `FramedRead` + `LengthDelimitedCodec`: an 8 KiB buffer for the connection's life, grown to the largest frame |
| The write buffer is allocated when a frame is buffered and dropped once all of it is written | `src/codec/framed_write.rs` | 16 KiB allocated at handshake, kept |
| The HPACK encoder uses no dynamic table (`DEFAULT_MAX_ALLOWED_SIZE = 0`) and does not keep its scratch buffer | `src/hpack/encoder.rs` | up to 4 KiB of table and the largest header block, kept |
| The HPACK decoder's Huffman scratch buffer is allocated on demand and dropped after each header block | `src/hpack/decoder.rs` | 4 KiB allocated at handshake, kept |
| Dynamic-table entries are copied into their own allocations | `src/hpack/decoder.rs`, `src/hpack/header.rs` | entries shared the frame's receive buffer and kept all of it alive |
| The stream store and the receive-frame buffer give back their memory when they become empty | `src/proto/streams/store.rs`, `src/proto/streams/buffer.rs` | kept the high-water mark of concurrent streams |
| A stream has no send capacity while data handed to it is still unsent: one batch in flight per stream, as nginx | `src/proto/streams/stream.rs` | capacity up to `max_send_buffer_size` bytes, so a stream fed 1-byte chunks while its client did not read queued up to that many frames (T004) |
| A non-CONNECT request without `:path` is malformed (stream reset) | `src/server.rs` | accepted, and its URI read as `/` (T115) |
