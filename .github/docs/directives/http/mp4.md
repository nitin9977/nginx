# Module ngx_http_mp4_module

**Source**: https://nginx.org/en/docs/http/ngx_http_mp4_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives mp4 mp4_buffer_size mp4_max_buffer_size mp4_limit_rate mp4_limit_rate_after mp4_start_key_frame The ngx_http_mp4_module module provides pseudo-streaming server-side support for MP4 files. Such files typically have the .mp4 , .m4v , or .m4a filename extensions. Pseudo-streaming works in alliance with a compatible media player. The player sends an HTTP request to the server with the start time specified in the query string argument (named simply start and specified in seconds), and the server responds with the stream such that its start position corresponds to the requested time, for example: http://example.com/elephants_dream.mp4?start=238.88 This allows performing a random seeking at any time, or starting playback in the middle of the timeline. To support s

## Example Configuration

```nginx
location /video/ {
    mp4;
    mp4_buffer_size       1m;
    mp4_max_buffer_size   5m;
    mp4_limit_rate        on;
    mp4_limit_rate_after  30s;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `mp4` | mp4 ; | — | location |
| `mp4_buffer_size` | mp4_buffer_size size ; | mp4_buffer_size 512K; | http , server , location |
| `mp4_max_buffer_size` | mp4_max_buffer_size size ; | mp4_max_buffer_size 10M; | http , server , location |
| `mp4_limit_rate` | mp4_limit_rate on \| off \| factor ; | mp4_limit_rate off; | http , server , location |
| `mp4_limit_rate_after` | mp4_limit_rate_after time ; | mp4_limit_rate_after 60s; | http , server , location |

## Directive Details

### `mp4`

**Syntax**: `mp4 ;`  
**Default**: `—`  
**Context**: `location`  

Turns on module processing in a surrounding location.

### `mp4_buffer_size`

**Syntax**: `mp4_buffer_size size ;`  
**Default**: `mp4_buffer_size 512K;`  
**Context**: `http , server , location`  

Sets the initial size of the buffer used for processing MP4 files.

### `mp4_max_buffer_size`

**Syntax**: `mp4_max_buffer_size size ;`  
**Default**: `mp4_max_buffer_size 10M;`  
**Context**: `http , server , location`  

During metadata processing, a larger buffer may become necessary. Its size cannot exceed the specified size , or else nginx will return the 500 (Internal Server Error) server error, and log the following message: "/some/movie/file.mp4" mp4 moov atom is too large: 12583268, you may want to increase mp4_max_buffer_size

### `mp4_limit_rate`

**Syntax**: `mp4_limit_rate on | off | factor ;`  
**Default**: `mp4_limit_rate off;`  
**Context**: `http , server , location`  

Limits the rate of response transmission to a client. The rate is limited based on the average bitrate of the MP4 file served. To calculate the rate, the bitrate is multiplied by the specified factor . The special value “ on ” corresponds to the factor of 1.1. The special value “ off ” disables rate limiting. The limit is set per a request, and so if a client simultaneously opens two connections, the overall rate will be twice as much as the specified limit.

### `mp4_limit_rate_after`

**Syntax**: `mp4_limit_rate_after time ;`  
**Default**: `mp4_limit_rate_after 60s;`  
**Context**: `http , server , location`  

Sets the initial amount of media data (measured in playback time) after which the further transmission of the response to a client will be rate limited.
