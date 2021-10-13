# Static Asset Caching

The `statics` sections expose static assets built into your application's container to Fly's Anycast network. When Fly's Anycast network handles requests for your application, it looks for static mappings and satisfies those requests directly from our proxy. This allows you to serve HTML files, Javascript, and images without needing to run a web server inside your container. 

You can have:
* No `statics` section: The application has no static asset caches.
* One `statics` section: The application has one mapping to a static asset cache.
* Multiple `statics` sections:  The application has up to 10 mappings to static asset caches.

The ``statics`` section is an array of tables in TOML. The section is marked with double square brackets:

```toml
[[statics]]
  guest_path = "app/public"
  url_prefix = "/static"
```

`guest_path`: The path where the files to serve are located in your container. You may refer multiple mappings to the same `guest_path`. **Please note:** Our static cache service does not currently honor symlinks. You must use the absolute original path in your configuration.

`url_prefix`: The URL path which will serve the files from the static asset cache. You must not use the same `url_prefix` for multiple mappings. You may have mappings with similar prefixes, such as `/static/foo` and `/static/bar`.

Responses from static asset caches include the `fly-cache-status` response header, with a value of `HIT` or `MISS`.