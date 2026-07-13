# nginx Directive Reference

Comprehensive reference for all nginx configuration directives, generated from
the official documentation at https://nginx.org/en/docs/.

---

## Introduction & Guides

| Document | Description |
|----------|-------------|
| [intro.md](./intro.md) | Config syntax, units, events, hashes, syslog, signals, QUIC, server names, request processing, load balancing, HTTPS |
| [core.md](./core.md) | `ngx_core_module` — daemon, worker_processes, error_log, include, events block |

---

## HTTP Modules

| File | Module | Key Directives |
|------|--------|----------------|
| [http/core.md](./http/core.md) | `ngx_http_core_module` | `server`, `location`, `listen`, `root`, `index`, `try_files`, `alias`, `types`, `keepalive_timeout` |
| [http/access.md](./http/access.md) | `ngx_http_access_module` | `allow`, `deny` |
| [http/addition.md](./http/addition.md) | `ngx_http_addition_module` | `add_before_body`, `add_after_body`, `addition_types` |
| [http/auth_basic.md](./http/auth_basic.md) | `ngx_http_auth_basic_module` | `auth_basic`, `auth_basic_user_file` |
| [http/auth_request.md](./http/auth_request.md) | `ngx_http_auth_request_module` | `auth_request`, `auth_request_set` |
| [http/autoindex.md](./http/autoindex.md) | `ngx_http_autoindex_module` | `autoindex`, `autoindex_exact_size`, `autoindex_format`, `autoindex_localtime` |
| [http/browser.md](./http/browser.md) | `ngx_http_browser_module` | `ancient_browser`, `modern_browser`, `modern_browser_value` |
| [http/charset.md](./http/charset.md) | `ngx_http_charset_module` | `charset`, `source_charset`, `charset_map`, `charset_types` |
| [http/dav.md](./http/dav.md) | `ngx_http_dav_module` | `dav_methods`, `dav_access`, `create_full_put_path`, `min_delete_depth` |
| [http/empty_gif.md](./http/empty_gif.md) | `ngx_http_empty_gif_module` | `empty_gif` |
| [http/fastcgi.md](./http/fastcgi.md) | `ngx_http_fastcgi_module` | `fastcgi_pass`, `fastcgi_param`, `fastcgi_index`, `fastcgi_cache`, `fastcgi_read_timeout` |
| [http/flv.md](./http/flv.md) | `ngx_http_flv_module` | `flv` |
| [http/geo.md](./http/geo.md) | `ngx_http_geo_module` | `geo` |
| [http/geoip.md](./http/geoip.md) | `ngx_http_geoip_module` | `geoip_country`, `geoip_city`, `geoip_org` |
| [http/grpc.md](./http/grpc.md) | `ngx_http_grpc_module` | `grpc_pass`, `grpc_bind`, `grpc_buffer_size`, `grpc_ssl_*` |
| [http/gunzip.md](./http/gunzip.md) | `ngx_http_gunzip_module` | `gunzip`, `gunzip_buffers` |
| [http/gzip.md](./http/gzip.md) | `ngx_http_gzip_module` | `gzip`, `gzip_comp_level`, `gzip_types`, `gzip_min_length`, `gzip_buffers` |
| [http/gzip_static.md](./http/gzip_static.md) | `ngx_http_gzip_static_module` | `gzip_static` |
| [http/headers.md](./http/headers.md) | `ngx_http_headers_module` | `add_header`, `add_trailer`, `expires` |
| [http/image_filter.md](./http/image_filter.md) | `ngx_http_image_filter_module` | `image_filter`, `image_filter_jpeg_quality`, `image_filter_webp_quality` |
| [http/index.md](./http/index.md) | `ngx_http_index_module` | `index` |
| [http/js.md](./http/js.md) | `ngx_http_js_module` | `js_import`, `js_content`, `js_set`, `js_header_filter`, `js_path`, `js_shared_dict_zone` |
| [http/limit_conn.md](./http/limit_conn.md) | `ngx_http_limit_conn_module` | `limit_conn_zone`, `limit_conn`, `limit_conn_status`, `limit_conn_log_level` |
| [http/limit_req.md](./http/limit_req.md) | `ngx_http_limit_req_module` | `limit_req_zone`, `limit_req`, `limit_req_burst`, `limit_req_status`, `limit_req_log_level` |
| [http/log.md](./http/log.md) | `ngx_http_log_module` | `access_log`, `log_format`, `open_log_file_cache` |
| [http/map.md](./http/map.md) | `ngx_http_map_module` | `map`, `map_hash_bucket_size`, `map_hash_max_size` |
| [http/memcached.md](./http/memcached.md) | `ngx_http_memcached_module` | `memcached_pass`, `memcached_bind`, `memcached_connect_timeout`, `memcached_read_timeout` |
| [http/mirror.md](./http/mirror.md) | `ngx_http_mirror_module` | `mirror`, `mirror_request_body` |
| [http/mp4.md](./http/mp4.md) | `ngx_http_mp4_module` | `mp4`, `mp4_buffer_size`, `mp4_max_buffer_size`, `mp4_start_key_frame` |
| [http/proxy.md](./http/proxy.md) | `ngx_http_proxy_module` | `proxy_pass`, `proxy_set_header`, `proxy_cache`, `proxy_cache_path`, `proxy_connect_timeout`, `proxy_read_timeout` |
| [http/random_index.md](./http/random_index.md) | `ngx_http_random_index_module` | `random_index` |
| [http/realip.md](./http/realip.md) | `ngx_http_realip_module` | `set_real_ip_from`, `real_ip_header`, `real_ip_recursive` |
| [http/referer.md](./http/referer.md) | `ngx_http_referer_module` | `valid_referers`, `referer_hash_bucket_size`, `referer_hash_max_size` |
| [http/rewrite.md](./http/rewrite.md) | `ngx_http_rewrite_module` | `rewrite`, `return`, `if`, `set`, `break`, `rewrite_log`, `uninitialized_variable_warn` |
| [http/scgi.md](./http/scgi.md) | `ngx_http_scgi_module` | `scgi_pass`, `scgi_param`, `scgi_cache`, `scgi_read_timeout` |
| [http/secure_link.md](./http/secure_link.md) | `ngx_http_secure_link_module` | `secure_link`, `secure_link_md5`, `secure_link_secret` |
| [http/slice.md](./http/slice.md) | `ngx_http_slice_module` | `slice` |
| [http/split_clients.md](./http/split_clients.md) | `ngx_http_split_clients_module` | `split_clients` |
| [http/ssi.md](./http/ssi.md) | `ngx_http_ssi_module` | `ssi`, `ssi_types`, `ssi_last_modified`, `ssi_min_file_chunk` |
| [http/ssl.md](./http/ssl.md) | `ngx_http_ssl_module` | `ssl_certificate`, `ssl_certificate_key`, `ssl_protocols`, `ssl_ciphers`, `ssl_session_cache`, `ssl_stapling` |
| [http/stub_status.md](./http/stub_status.md) | `ngx_http_stub_status_module` | `stub_status` |
| [http/sub.md](./http/sub.md) | `ngx_http_sub_module` | `sub_filter`, `sub_filter_once`, `sub_filter_types`, `sub_filter_last_modified` |
| [http/upstream.md](./http/upstream.md) | `ngx_http_upstream_module` | `upstream`, `server`, `keepalive`, `keepalive_requests`, `hash`, `ip_hash`, `least_conn`, `zone` |
| [http/upstream_hc.md](./http/upstream_hc.md) | `ngx_http_upstream_hc_module` | `health_check`, `match` |
| [http/userid.md](./http/userid.md) | `ngx_http_userid_module` | `userid`, `userid_name`, `userid_domain`, `userid_path`, `userid_expires` |
| [http/uwsgi.md](./http/uwsgi.md) | `ngx_http_uwsgi_module` | `uwsgi_pass`, `uwsgi_param`, `uwsgi_cache`, `uwsgi_read_timeout` |
| [http/v2.md](./http/v2.md) | `ngx_http_v2_module` | `http2`, `http2_push`, `http2_push_preload`, `http2_chunk_size`, `http2_max_concurrent_streams` |
| [http/v3.md](./http/v3.md) | `ngx_http_v3_module` | `http3`, `http3_max_concurrent_streams`, `quic_retry`, `quic_gso` |
| [http/xslt.md](./http/xslt.md) | `ngx_http_xslt_module` | `xslt_stylesheet`, `xslt_types`, `xslt_last_modified` |

---

## Stream Modules (TCP/UDP Proxy)

| File | Module | Key Directives |
|------|--------|----------------|
| [stream/core.md](./stream/core.md) | `ngx_stream_core_module` | `stream`, `server`, `listen`, `resolver`, `error_log`, `worker_processes` |
| [stream/access.md](./stream/access.md) | `ngx_stream_access_module` | `allow`, `deny` |
| [stream/geo.md](./stream/geo.md) | `ngx_stream_geo_module` | `geo` |
| [stream/geoip.md](./stream/geoip.md) | `ngx_stream_geoip_module` | `geoip_country`, `geoip_city` |
| [stream/js.md](./stream/js.md) | `ngx_stream_js_module` | `js_import`, `js_access`, `js_filter`, `js_preread`, `js_set` |
| [stream/limit_conn.md](./stream/limit_conn.md) | `ngx_stream_limit_conn_module` | `limit_conn_zone`, `limit_conn`, `limit_conn_status` |
| [stream/log.md](./stream/log.md) | `ngx_stream_log_module` | `access_log`, `log_format`, `open_log_file_cache` |
| [stream/map.md](./stream/map.md) | `ngx_stream_map_module` | `map` |
| [stream/proxy.md](./stream/proxy.md) | `ngx_stream_proxy_module` | `proxy_pass`, `proxy_bind`, `proxy_connect_timeout`, `proxy_timeout`, `proxy_ssl` |
| [stream/realip.md](./stream/realip.md) | `ngx_stream_realip_module` | `set_real_ip_from`, `real_ip_header` |
| [stream/return.md](./stream/return.md) | `ngx_stream_return_module` | `return` |
| [stream/set.md](./stream/set.md) | `ngx_stream_set_module` | `set` |
| [stream/split_clients.md](./stream/split_clients.md) | `ngx_stream_split_clients_module` | `split_clients` |
| [stream/ssl.md](./stream/ssl.md) | `ngx_stream_ssl_module` | `ssl_certificate`, `ssl_protocols`, `ssl_ciphers`, `ssl_session_cache`, `ssl_verify_client` |
| [stream/ssl_preread.md](./stream/ssl_preread.md) | `ngx_stream_ssl_preread_module` | `ssl_preread`, `$ssl_preread_server_name`, `$ssl_preread_alpn_protocols` |
| [stream/upstream.md](./stream/upstream.md) | `ngx_stream_upstream_module` | `upstream`, `server`, `zone`, `hash`, `least_conn`, `least_time` |
| [stream/upstream_hc.md](./stream/upstream_hc.md) | `ngx_stream_upstream_hc_module` | `health_check`, `match` |

---

## Mail Modules (SMTP/IMAP/POP3 Proxy)

| File | Module | Key Directives |
|------|--------|----------------|
| [mail/core.md](./mail/core.md) | `ngx_mail_core_module` | `mail`, `server`, `listen`, `protocol`, `resolver` |
| [mail/auth_http.md](./mail/auth_http.md) | `ngx_mail_auth_http_module` | `auth_http`, `auth_http_header`, `auth_http_timeout` |
| [mail/proxy.md](./mail/proxy.md) | `ngx_mail_proxy_module` | `proxy_buffer`, `proxy_timeout`, `proxy_pass_error_message` |
| [mail/ssl.md](./mail/ssl.md) | `ngx_mail_ssl_module` | `ssl_certificate`, `ssl_protocols`, `ssl_ciphers`, `starttls` |
| [mail/imap.md](./mail/imap.md) | `ngx_mail_imap_module` | `imap_auth`, `imap_capabilities`, `imap_client_buffer` |
| [mail/pop3.md](./mail/pop3.md) | `ngx_mail_pop3_module` | `pop3_auth`, `pop3_capabilities` |
| [mail/smtp.md](./mail/smtp.md) | `ngx_mail_smtp_module` | `smtp_auth`, `smtp_capabilities`, `smtp_client_buffer`, `smtp_greeting_delay` |

---

## Other Modules

| File | Module | Key Directives |
|------|--------|----------------|
| [google_perftools.md](./google_perftools.md) | `ngx_google_perftools_module` | `google_perftools_profiles` |
| [otel.md](./otel.md) | `ngx_otel_module` | `otel_exporter`, `otel_service_name`, `otel_trace`, `otel_span_name`, `otel_trace_context` |

---

## Quick Reference — Most Used Directives

### Serving Static Files
```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/html;
    index index.html;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```
→ See [http/core.md](./http/core.md)

### Reverse Proxy
```nginx
upstream backend {
    server 127.0.0.1:8080;
    keepalive 32;
}
location / {
    proxy_pass http://backend;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_connect_timeout 5s;
    proxy_read_timeout 60s;
}
```
→ See [http/proxy.md](./http/proxy.md), [http/upstream.md](./http/upstream.md)

### SSL/TLS
```nginx
server {
    listen 443 ssl;
    ssl_certificate     /etc/nginx/cert.pem;
    ssl_certificate_key /etc/nginx/cert.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;
}
```
→ See [http/ssl.md](./http/ssl.md)

### Rate Limiting
```nginx
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    server {
        location /api/ {
            limit_req zone=api burst=20 nodelay;
        }
    }
}
```
→ See [http/limit_req.md](./http/limit_req.md)

### Access Control
```nginx
location /admin {
    allow 10.0.0.0/8;
    deny  all;
}
```
→ See [http/access.md](./http/access.md)

### Gzip Compression
```nginx
gzip on;
gzip_comp_level 6;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1024;
```
→ See [http/gzip.md](./http/gzip.md)

### FastCGI (PHP)
```nginx
location ~ \.php$ {
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
}
```
→ See [http/fastcgi.md](./http/fastcgi.md)
