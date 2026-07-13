# Module ngx_http_image_filter_module

**Source**: https://nginx.org/en/docs/http/ngx_http_image_filter_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives image_filter image_filter_buffer image_filter_interlace image_filter_jpeg_quality image_filter_sharpen image_filter_transparency image_filter_webp_quality The ngx_http_image_filter_module module (0.7.54+) is a filter that transforms images in JPEG, GIF, PNG, and WebP formats. This module is not built by default, it should be enabled with the --with-http_image_filter_module configuration parameter. This module utilizes the libgd library. It is recommended to use the latest available version of the library. The WebP format support appeared in version 1.11.6. To transform images in this format, the libgd library must be compiled with the WebP support.

## Example Configuration

```nginx
location /img/ {
    proxy_pass   http://backend;
    image_filter resize 150 100;
    image_filter rotate 90;
    error_page   415 = /empty;
}

location = /empty {
    empty_gif;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `image_filter` | image_filter off ; image_filter test ; image_filter size ; image_filter rotate 90 \| 180 \| 270 ; image_filter resize wi | image_filter off; | location |
| `image_filter_buffer` | image_filter_buffer size ; | image_filter_buffer 1M; | http , server , location |
| `image_filter_jpeg_quality` | image_filter_jpeg_quality quality ; | image_filter_jpeg_quality 75; | http , server , location |
| `image_filter_sharpen` | image_filter_sharpen percent ; | image_filter_sharpen 0; | http , server , location |
| `image_filter_transparency` | image_filter_transparency on \| off ; | image_filter_transparency on; | http , server , location |

## Directive Details

### `image_filter`

**Syntax**: `image_filter off ; image_filter test ; image_filter size ; image_filter rotate 90 | 180 | 270 ; image_filter resize width height ; image_filter crop width height ;`  
**Default**: `image_filter off;`  
**Context**: `location`  

Sets the type of transformation to perform on images: { "img" : { "width": 100, "height": 100, "type": "gif" } } In case of an error, the output is as follows: {} rotate 90 | 180 | 270 rotates images counter-clockwise by the specified number of degrees. Parameter value can contain variables. This mode can be used either alone or along with the resize and crop transformations. resize width height proportionally reduces an image to the specified sizes. To reduce by only one dimension, another dimension can be specified as “ - ”. In case of an error, the server will return code 415 (Unsupported Media Type). Parameter values can contain variables. When used along with the rotate parameter, the rotation happens after reduction. crop width height proportionally reduces an image to the larger side size and crops extraneous edges by another side. To reduce by only one dimension, another dimension can be specified as “ - ”. In case of an error, the server will return code 415 (Unsupported Media Type). Parameter values can contain variables. When used along with the rotate parameter, the rotation happens before reduction.

### `image_filter_buffer`

**Syntax**: `image_filter_buffer size ;`  
**Default**: `image_filter_buffer 1M;`  
**Context**: `http , server , location`  

Sets the maximum size of the buffer used for reading images. When the size is exceeded the server returns error 415 (Unsupported Media Type).

### `image_filter_jpeg_quality`

**Syntax**: `image_filter_jpeg_quality quality ;`  
**Default**: `image_filter_jpeg_quality 75;`  
**Context**: `http , server , location`  

Sets the desired quality of the transformed JPEG images. Acceptable values are in the range from 1 to 100. Lesser values usually imply both lower image quality and less data to transfer. The maximum recommended value is 95. Parameter value can contain variables.

### `image_filter_sharpen`

**Syntax**: `image_filter_sharpen percent ;`  
**Default**: `image_filter_sharpen 0;`  
**Context**: `http , server , location`  

Increases sharpness of the final image. The sharpness percentage can exceed 100. The zero value disables sharpening. Parameter value can contain variables.

### `image_filter_transparency`

**Syntax**: `image_filter_transparency on | off ;`  
**Default**: `image_filter_transparency on;`  
**Context**: `http , server , location`  

Defines whether transparency should be preserved when transforming GIF images or PNG images with colors specified by a palette. The loss of transparency results in images of a better quality. The alpha channel transparency in PNG is always preserved.
