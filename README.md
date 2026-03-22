# varnish-urlsort

`varnish-urlsort` normalizes URL query strings for Varnish by sorting query parameters into a stable order before the request is used as a cache key.

## Why this exists

Many applications generate the same request with query parameters in different orders:

- `/search?a=1&b=2`
- `/search?b=2&a=1`

Those URLs are usually equivalent at the application layer, but Varnish treats them as different cache keys unless the URL is normalized first. That fragments the cache, lowers hit rate, and increases backend traffic.

This project solves that by rewriting:

```text
/search?b=2&a=1
```

into:

```text
/search?a=1&b=2
```

Equivalent requests then converge on the same cache object.

## Project layout

- `urlsort.c` contains the core query-string sorting logic.
- `main/` builds a small CLI wrapper around the same C code.
- `vmod/` packages the sorter as an out-of-tree Varnish VMOD.
- `urlsort.vcl` shows the VMOD-based integration.
- `urlsort_inline.vcl` shows the original inline-C VCL integration.
- `url_load_test.rb` is a simple load generator for exercising repeated requests with shuffled parameter order.

## How it works

The implementation splits the URL at `?`, tokenizes the query string on `&`, inserts each parameter into a binary tree, and emits the parameters back in lexical order. The path portion of the URL is preserved and only the query-string ordering changes.

The important idea is not "sorting for its own sake"; it is cache-key canonicalization. If the origin treats parameter order as insignificant, sorting produces a single canonical representation that Varnish can cache effectively.

## When to use it

Use this when:

- your application treats query parameter order as irrelevant
- the same logical URL is generated in multiple parameter orders
- you want better Varnish hit ratios for query-string-heavy traffic

Do not use this when parameter order is semantically meaningful for the backend.

## Integration options

### VMOD integration

The VMOD approach is the cleaner integration for Varnish deployments:

```vcl
import urlsort;

sub vcl_recv {
  set req.url = urlsort.sortquery(req.url);
}
```

The sample VCL is included in [`urlsort.vcl`](urlsort.vcl).

### Inline C integration

For older setups that embed C directly in VCL, the repository also includes an inline version:

```vcl
C{
#include "../urlsort.c"
}C

sub vcl_recv {
    C{
       const char *url = VRT_r_req_url(sp);
       char *urldup = strdup(url);
       char *sorted = urlsort(urldup);
       VRT_l_req_url(sp, sorted, vrt_magic_string_end);
    }C
}
```

See [`urlsort_inline.vcl`](urlsort_inline.vcl).

## Building

### CLI

The CLI in `main/` is useful for quick verification of the core sorter:

```sh
cd main
cmake .
make
./urlsort '/search?c=3&a=1&b=2'
```

### VMOD

The VMOD uses autotools and expects a Varnish source tree or headers compatible with the target installation:

```sh
cd vmod
./autogen.sh
./configure VARNISHSRC=/path/to/varnish/source [VMODDIR=/path/to/vmods]
make
make check
sudo make install
```

The exact include and install paths depend on the Varnish version you are targeting.

## Testing the idea

`url_load_test.rb` sends repeated requests with the same parameters shuffled into different orders. It is a simple way to observe whether URL normalization improves cache reuse in front of an application that treats those URLs as equivalent.

## Caveats

- This project assumes parameter order is not meaningful.
- It canonicalizes ordering only; it does not attempt broader URL normalization.
- The VCL examples in this repository reflect older Varnish APIs and may need adjustment for newer releases.

## License

Released under the BSD 2-Clause license. See [`LICENSE`](LICENSE).
