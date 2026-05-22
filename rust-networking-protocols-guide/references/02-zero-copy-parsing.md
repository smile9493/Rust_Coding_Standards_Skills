# 02 — Zero-Copy Wire Parsing

**Status**: Core  
**Prerequisites**: `nom` ≥ 7 / `winnow` ≥ 0.6

---

## Overview

Network protocol parsers process untrusted bytes at wire speed. Allocating `Vec<u8>` copies of every parsed field is a performance bottleneck and a DoS vector. Zero-copy parsing operates directly on `&[u8]` slices, returning references into the input buffer. `nom` and `winnow` are combinator-based parser frameworks designed for this pattern.

---

## 1. nom — Combinator Parsing on `&[u8]`

nom combinators compose small parsers into complete protocol decoders. The output is a `nom::IResult<&[u8], Parsed>` where the first element is the remaining unparsed input.

```rust
use nom::{
    bytes::streaming::{tag, take},
    combinator::{map, map_res},
    multi::length_data,
    number::streaming::{be_u16, be_u32},
    sequence::tuple,
    IResult,
};

#[derive(Debug, PartialEq)]
struct DnsHeader {
    id: u16,
    flags: u16,
    qdcount: u16,
    ancount: u16,
    nscount: u16,
    arcount: u16,
}

#[derive(Debug, PartialEq)]
struct DnsQuestion<'a> {
    qname: &'a [u8],
    qtype: u16,
    qclass: u16,
}

#[derive(Debug, PartialEq)]
struct DnsMessage<'a> {
    header: DnsHeader,
    questions: Vec<DnsQuestion<'a>>,
}

fn parse_dns_header(input: &[u8]) -> IResult<&[u8], DnsHeader> {
    let (input, (id, flags, qdcount, ancount, nscount, arcount)) = tuple((
        be_u16, be_u16, be_u16, be_u16, be_u16, be_u16,
    ))(input)?;

    Ok((
        input,
        DnsHeader { id, flags, qdcount, ancount, nscount, arcount },
    ))
}

fn parse_dns_label(input: &[u8]) -> IResult<&[u8], &[u8]> {
    let (input, len) = take(1usize)(input)?;
    let len = len[0] as usize;
    if len == 0 {
        return Ok((input, &[]));
    }
    take(len)(input)
}

fn parse_qname(input: &[u8]) -> IResult<&[u8], &[u8]> {
    let start = input;
    let mut input = input;
    loop {
        let (rest, label) = parse_dns_label(input)?;
        if label.is_empty() {
            let consumed = input.as_ptr() as usize - start.as_ptr() as usize + 1;
            return Ok((&start[consumed..], &start[..consumed]));
        }
        input = rest;
    }
}

fn parse_question(input: &[u8]) -> IResult<&[u8], DnsQuestion> {
    let (input, qname) = parse_qname(input)?;
    let (input, (qtype, qclass)) = tuple((be_u16, be_u16))(input)?;
    Ok((input, DnsQuestion { qname, qtype, qclass }))
}

fn parse_dns_message(input: &[u8]) -> IResult<&[u8], DnsMessage> {
    let (input, header) = parse_dns_header(input)?;
    let (input, questions) =
        nom::multi::count(parse_question, header.qdcount as usize)(input)?;
    Ok((input, DnsMessage { header, questions }))
}
```

---

## 2. winnow — Streaming Parsers for Incomplete Input

`winnow` is the successor to nom with better error messages and a streaming mode via `winnow::Partial`. Streaming parsers return `Incomplete` when more data is needed, enabling incremental processing over TCP byte streams.

```rust
use winnow::{
    binary::streaming::{be_u16, be_u32, take},
    combinator::{repeat, trace},
    error::{ContextError, ErrMode},
    prelude::*,
    stream::Partial,
    token::tag,
};

#[derive(Debug)]
struct TlsRecordHeader {
    content_type: u8,
    version: u16,
    length: u16,
}

fn parse_tls_record<'a>(
    input: &mut Partial<&'a [u8]>,
) -> PResult<TlsRecordHeader, ContextError> {
    trace(
        "tls_record_header",
        (
            take(1usize).map(|b: &[u8]| b[0]),
            be_u16,
            be_u16,
        ).map(|(content_type, version, length)| TlsRecordHeader {
            content_type,
            version,
            length,
        }),
    )
    .parse_next(input)
}

fn try_parse_tls_stream(buf: &mut Partial<&[u8]>) -> Result<TlsRecordHeader, ErrMode<ContextError>> {
    let header = parse_tls_record(buf)?;
    if buf.len() < header.length as usize {
        return Err(ErrMode::Incomplete(winnow::error::Needed::Size(
            std::num::NonZero::new(header.length as usize).unwrap(),
        )));
    }
    buf.advance(header.length as usize);
    Ok(header)
}
```

---

## 3. Binary Protocol Parsing — Fixed-Width Headers

Fixed-width binary headers are the most common pattern. Parse the header first, then use the length field to determine how many payload bytes to consume.

```rust
use nom::IResult;
use nom::number::streaming::{be_u16, be_u32, be_u8};

#[derive(Debug)]
struct QuicLongHeader {
    version: u32,
    dst_conn_id_len: u8,
    dst_conn_id: u64,
    src_conn_id_len: u8,
    src_conn_id: u64,
}

fn parse_quic_long_header(input: &[u8]) -> IResult<&[u8], QuicLongHeader> {
    let (input, first_byte) = be_u8(input)?;
    let is_long_header = first_byte & 0x80 != 0;
    if !is_long_header {
        return Err(nom::Err::Error(nom::error::Error::new(
            input,
            nom::error::ErrorKind::Tag,
        )));
    }

    let (input, version) = be_u32(input)?;
    let (input, dst_len_raw) = be_u8(input)?;
    let dst_conn_id_len = dst_len_raw;

    let (input, dst_bytes) = nom::bytes::streaming::take(dst_conn_id_len as usize)(input)?;
    let mut dst_buf = [0u8; 8];
    dst_buf[..dst_conn_id_len as usize].copy_from_slice(dst_bytes);
    let dst_conn_id = u64::from_be_bytes(dst_buf);

    let (input, src_len_raw) = be_u8(input)?;
    let src_conn_id_len = src_len_raw;

    let (input, src_bytes) = nom::bytes::streaming::take(src_conn_id_len as usize)(input)?;
    let mut src_buf = [0u8; 8];
    src_buf[..src_conn_id_len as usize].copy_from_slice(src_bytes);
    let src_conn_id = u64::from_be_bytes(src_buf);

    Ok((
        input,
        QuicLongHeader {
            version,
            dst_conn_id_len,
            dst_conn_id,
            src_conn_id_len,
            src_conn_id,
        },
    ))
}
```

---

## 4. Zero-Copy by Design — References, Not Copies

Every field returned from a parser should be a reference into the original buffer, not a heap allocation. Compare the two approaches:

```rust
use bytes::Bytes;

struct BadParser;
struct GoodParser;

impl BadParser {
    fn parse_header(input: &[u8]) -> Result<CopiedHeader, ()> {
        if input.len() < 12 { return Err(()); }
        Ok(CopiedHeader {
            method: String::from_utf8_lossy(&input[0..4]).to_string(),
            path: String::from_utf8_lossy(&input[4..12]).to_string(),
        })
    }
}

struct CopiedHeader {
    method: String,
    path: String,
}

impl GoodParser {
    fn parse_header<'a>(input: &'a [u8]) -> Result<BorrowedHeader<'a>, ()> {
        if input.len() < 12 { return Err(()); }
        Ok(BorrowedHeader {
            method: &input[0..4],
            path: &input[4..12],
        })
    }
}

struct BorrowedHeader<'a> {
    method: &'a [u8],
    path: &'a [u8],
}

fn decode_request(input: &[u8], _total_len: usize) -> Result<(), ()> {
    let bad = BadParser::parse_header(input)?;
    let _method: String = bad.method;

    let good = GoodParser::parse_header(input)?;
    let method_str = std::str::from_utf8(good.method).map_err(|_| ())?;
    let path_str = std::str::from_utf8(good.path).map_err(|_| ())?;

    assert_eq!(method_str, "GET ");
    assert_eq!(path_str, "/index\0\0");
    Ok(())
}
```

---

## 5. Verifying with Checksums and Lengths

Parsing must verify integrity. Checksum validation and length-prefix verification are mandatory before trusting any parsed data.

```rust
fn verify_ipv4_checksum(header: &[u8; 20]) -> u16 {
    let mut sum: u32 = 0;
    for i in (0..20).step_by(2) {
        sum += u16::from_be_bytes([header[i], header[i + 1]]) as u32;
    }
    while sum >> 16 != 0 {
        sum = (sum & 0xFFFF) + (sum >> 16);
    }
    !(sum as u16)
}

fn parse_and_verify_ipv4(input: &[u8]) -> Result<Ipv4Parsed, ParseError> {
    if input.len() < 20 {
        return Err(ParseError::Truncated);
    }
    let header: [u8; 20] = input[..20].try_into().unwrap();

    let checksum = verify_ipv4_checksum(&header);
    let declared = u16::from_be_bytes([header[10], header[11]]);
    if checksum != declared {
        return Err(ParseError::ChecksumMismatch);
    }

    let total_len = u16::from_be_bytes([header[2], header[3]]) as usize;
    if total_len < 20 || total_len > input.len() {
        return Err(ParseError::LengthOutOfRange);
    }

    Ok(Ipv4Parsed {
        src: [header[12], header[13], header[14], header[15]],
        dst: [header[16], header[17], header[18], header[19]],
        protocol: header[9],
        payload_offset: 20,
        payload_len: total_len - 20,
    })
}

struct Ipv4Parsed {
    src: [u8; 4],
    dst: [u8; 4],
    protocol: u8,
    payload_offset: usize,
    payload_len: usize,
}

#[derive(Debug)]
enum ParseError {
    Truncated,
    ChecksumMismatch,
    LengthOutOfRange,
}
```

---

## Red Lines

1. Never allocate a `Vec<u8>` or `String` for parsed fields. Return `&[u8]` or `&str` references into the input buffer.
2. Streaming parsers must handle `Incomplete` gracefully — buffer and retry when more data arrives. Never discard partial input.
3. All length-prefixed fields must be bounds-checked before consuming. A malicious length field must not cause a panic or OOM.
4. Checksums and integrity fields must be verified at parse time, not deferred. An unverified checksum is equivalent to trusting the wire.
5. Prefer `winnow` for new projects. Its error reporting is superior and it is actively maintained. Use `nom` if you need a mature ecosystem.

---

## References

- [winnow crate](https://docs.rs/winnow/latest/winnow/) — streaming/combinator parser
- [nom crate](https://docs.rs/nom/latest/nom/) — original combinator parser framework
- [RFC 1035 — DNS Wire Format](https://datatracker.ietf.org/doc/html/rfc1035)
- [RFC 8446 §5 — TLS Record Layer](https://datatracker.ietf.org/doc/html/rfc8446#section-5)
- [RFC 9000 §17 — QUIC Long Header](https://datatracker.ietf.org/doc/html/rfc9000#section-17)