---
title: Caching - Advanced Topics
description: Learn advanced details about cached and authentication.
category:
  - developing

---

## Allow a User to Bypass the Cache

Pantheon supports setting a NO\_CACHE cookie for users who should bypass the cache. When this cookie is present, Varnish will neither get the user's response from any existing cache or store the response from the user into the cache.

This allows users to immediately see comments or changes they've made, even if they're not logged in. To best achieve this effect, we recommend setting the NO\_CACHE cookie to exist slightly longer than the site's page cache. This setting allows content contributors to resume using the cached pages once all cached pages have been updated.


## Ignoring GET Parameters

For the purposes of caching, Varnish ignores any GET parameter that is prefixed with two underscores to be compatible with services such as AdWords. The double-underscore prefix for params and cookies which can be ignored by the backend is an emerging standard.

For example, <tt>?__dynamic_id=1234</tt> would be ignored, but <tt>?dynamic_id=1234</tt> and <tt>?_dynamic_id</tt> would be considered distinct pages.

Query keys will still be passed to the application server, but the values will be changed to PANTHEON\_STRIPPED to indicate that the URL is being altered. For more information, see [PANTHEON\_STRIPPED parameters](/docs/articles/architecture/edge/pantheon_stripped-get-parameter-values).


## External Authentication (e.g. Facebook login)

If your site or application requires Facebook authentication, we have added exceptions for this to allow users to register and login. In the event you are having problems with another external authentication service, please contact us and let us know what service you are having issues with.


## Using Your Own Session-Style Cookies

Pantheon passes all cookies beginning with SESS that are followed by numbers and lowercase characters back to the application. When at least one of these cookies is present, Varnish will not try to respond to the request from its cache or store the response.


### Drupal Sites
Drupal uses SESS-prefixed cookies for its own session tracking, so be sure to name yours differently if you choose to use one. Generally, SESS followed by a few words will work.

**Correct:** SESSmysessioncookie, SESShello123, SESSletsgo

**Incorrect:** SESS\_hello, SESS-12345, mycustomSESS, Sessone, sess123testing, SESSFIVE


### WordPress Sites
WordPress does not use PHP session cookies; however, some themes and plugins do. If you are using a theme or plugin that requires PHP sessions, you can install [Pantheon-sessions](https://wordpress.org/plugins/wp-native-php-sessions/ "Panthon Session WordPress plugin"). It is designed to handle the naming properly.


## Geolocation, Referral Tracking, Content Customization, and Cache Segmentation Using STYXKEY

A site may need to deliver different content to different users without them logging in or starting a full session (either of which will cause them to bypass the page cache entirely). Pantheon supports this by allowing sites to set a cookie beginning with `STYXKEY` followed by one or more alphanumeric characters, hyphens, or underscores.

For example, you could set a cookie named `STYXKEY-country` to `ca` or `de` and cache different page content for each country. A site can have any number of `STYXKEY` cookies for varying content. 

**Examples of `STYXKEY` cookie names:**

&#8211; `STYXKEY-mobile-ios`: Delivers different stylesheets and content for iOS devices

&#8211; `STYXKEY_european_user`: Presents different privacy options to E.U. users

&#8211; `STYXKEY-under21`: Part of your site markets alcohol and you want to change the content for minors

&#8211; `STYXKEY-school`: Your site changes content depending on the user's school affiliation

**Invalid names that won't work:**

&#8211; `STYXKEY`: Needs something after the `STYXKEY` text

&#8211; `styxkey-android`: The text `STYXKEY` must be uppercase

&#8211; `STYX-KEY-android`: The text `STYXKEY` cannot be hyphenated or contain other punctuation

&#8211; `STYXKEY.tablet`: The only valid characters are a-z, A-Z, 0-9, hyphens ("-"), and underscores ("\_")

&#8211; `tablet-STYXKEY`: The cookie name must start with `STYXKEY`


## Varnish Servers

Pantheon uses a rotating pool of Varnish servers. Varnish does not have a shared pool or cache, so that means there is a distinct cache for each server. While local DNS typically picks a route and keeps using it, it is possible to access a different Varnish server and experience a cache miss.

The Max-Age returned in the header may vary depending on which cache server is hit. The main concern when examining Age is whether or not it is increasing, as this indicates that Varnish is indeed working.


## Varnish, Public Files, and Cookies

Pantheon strips cookies from requests made to public files served from sites/default/files, which allows Varnish to cache the response.


## SSL & Varnish

When a Pantheon environment is configured with SSL, a dedicated IP address to a load balancer is provided. Connections via SSL to the load balancer are decrypted by an SSL termination server using the client’s uploaded certificate, then handled like any other request, including the same rules for Varnish caching. The result is encrypted by the SSL termination server and served back to the client, completing the request.


## 404s & Varnish

Pantheon’s default is to not cache 404s, but if your application sets Cache-Control:max-age headers, Varnish will respect them. Depending on your use case, that may be the desired result.


### Drupal Sites

Drupal’s 404\_fast\_\* configuration does not set caching headers. Some contributed 404 modules include cache-friendly headers, which will cause a 404 response to be cached.


### WordPress Sites

WordPress does not by default set cache headers, 404 or otherwise. If your site has a Permalinks option set other than defauly, WordPress will return your theme's 404 page. Unless a plugin sets cache friendly headers, your 404 page will not be cached.


## Basic Authentication & Varnish

If you're using the Environment Access: Locked security setting on a site environment, Varnish will not cache your content.


## Purging URLs from Varnish

Pantheon supports purging of individual URLs from the Varnish edge cache through an API call. This is useful in situations where you want to selectively clear paths without flushing your site's entire cache. With this technique, it is possible to set very long cache lifetimes in general, and then take control over managing the edge cache in your code. It requires upfront knowledge of where your content appears on your site, as it cannot automatically discover all the places where the cache might need to be purged. Therefore, it is recommended for power users and very high traffic sites only.

### Example

Our example is a site that hosts a blog, and displays blog posts individually. It also has a list of "Recent blog posts", which displays 10 posts per page and uses the query string `?page=x` to navigate them. The site also displays slightly different markup for users in Europe, to comply with the local laws, using a special `STYXKEY` cookie (explained in the "Using STYXKEY" section above).

An editor modifies the blog post at `/blog/2015-03-20-my-blog-post`. Varnish has an existing copy of that page's content in its cache, as well as the listing page, `/recent-blog-posts`, on **page 4**. To selectively purge all the old versions of this blog post then, the following PHP code would need to be executed when the editor clicks "Save":

    $paths = array(
      '/blog/2015-03-20-my-blog-post',
      '/recent-blog-posts?page=4',
    );
    $cookies = array(
      '', // Represents "no cookie present".
      'STYXKEY_european_user=yes'
    );
    try {
      pantheon_purge_edge_urls($paths, $cookies);
    }
    catch (Exception $e) {
      // Do some error handling.
    }

Assuming the content is available on the hostname `example.com`, the following would be purged:

    http://example.com/blog/2015-03-20-my-blog-post []
    http://example.com/recent-blog-posts?page=4 []
    http://example.com/blog/2015-03-20-my-blog-post [STYXKEY_european_user=yes]
    http://example.com/recent-blog-posts?page=4 [STYXKEY_european_user=yes]

As you can see, keeping track of which URLs need to be purged is a non-trivial task; the method of computing this list is left up to your individual implementation.

### Usage documentation

`function pantheon_purge_edge_urls($paths, $hostnames = NULL, $cookies = NULL, $https = NULL)`

Send a PURGE request to the Varnish edge cache, for every combination of hostname, path, and cookies given as parameters.

There is a limit to how much content you may purge in a single call. That limit is **10**, calculated by multiplying `$hostnames` x `$paths` x `$cookies`. If the limit is exceeded, an `UnexpectedValueException` will be thrown.

There is no wildcard support at this time.

- `$paths`
  - Required. Array of paths, or a single string path. Include query string.
  - Example: `array('/', '/blogs', '/otherpath?key=value')`
- `$cookies`
  - Optional. Array of cookies, or single string cookie, or `NULL`.
  - All combinations of cookies, hostnames and paths will be purged.
    If `NULL` or `''`, no cookie will be sent.
  - Example: `array('STYXKEY123=on', '')`
- `$hostnames`
  - Optional. Array of hostnames, or single string hostname, or `NULL`.
    If `NULL`, the hostname will be taken from the current request's $_SERVER['HTTP_HOST'] value.
  - Example: `array('example.com', 'eu.example.com')`
- `$https`
  - Optional boolean. If `TRUE`, the https version of the content is purged,
    else the plain http version. Defaults to `NULL`, which does autodetect based
    on the current request's scheme.
- `@return`
  - An object with properties `code` and `body`. The code is the response code,
    `200` means OK. The `body` contains a message detailing the action taken, for
    example:
      `Purged Varnish cache for hosts: ['x'], paths: ['/'], cookies: [STYXKEY123=on'], https: False`
- `@exceptions`
  - Throws `InvalidArgumentException` if bad arguments are passed, and `UnexpectedValueException` in the case of API or other errors.


## Pantheon's Varnish cookie handling

Advanced Drupal and WordPress developers should reference this if they have any questions regarding what Pantheon Varnish does or does not cache. When any of the cookies below are present, Varnish will neither get the response from any existing cache, or store the response from the backend into the cache:

    NO_CACHE=
    S+ESS[a-z0-9]+=
    fbs[a-z0-9_]+=
    SimpleSAML[A-Za-z]+=
    SimpleSAML[A-Za-z]+=
    PHPSESSID=
    wordpress[A-Za-z0-9_]*=
    wp-[A-Za-z0-9_]+=
    comment_author_[a-z0-9_]+=
    duo_wordpress_auth_cookie=
    duo_secure_wordpress_auth_cookie=
    bp_completed_create_steps=
    bp_new_group_id=
    wp-resetpass-[A-Za-z0-9_]+=
    (wp_)?woocommerce[A-Za-z0-9_-]+=

The following cookies will cause Varnish to store different copies of the content for each different value:

    STYXKEY[a-zA-Z0-9-_]+=
    Drupal[a-zA-Z0-9-_\.]+=
