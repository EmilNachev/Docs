---
title: Response Caching Middleware in ASP.NET Core
author: guardrex
description: Learn how to configure and use Response Caching Middleware in ASP.NET Core.
manager: wpickett
monikerRange: '>= aspnetcore-1.1'
ms.author: riande
ms.custom: mvc
ms.date: 01/26/2017
ms.prod: asp.net-core
ms.topic: article
uid: performance/caching/middleware
---
# Response Caching Middleware in ASP.NET Core

By [Luke Latham](https://github.com/guardrex) and [John Luo](https://github.com/JunTaoLuo)

[View or download ASP.NET Core 2.1 sample code](https://github.com/aspnet/Docs/tree/master/aspnetcore/performance/caching/middleware/samples) ([how to download](xref:tutorials/index#how-to-download-a-sample))

This article explains how to configure Response Caching Middleware in an ASP.NET Core app. The middleware determines when responses are cacheable, stores responses, and serves responses from cache. For an introduction to HTTP caching and the `ResponseCache` attribute, see [Response Caching](xref:performance/caching/response).

## Package

To include the middleware in your project, add a reference to the [Microsoft.AspNetCore.ResponseCompression](https://www.nuget.org/packages/Microsoft.AspNetCore.ResponseCompression/) package or use the [Microsoft.AspNetCore.App metapackage](xref:fundamentals/metapackage-app), which is available for use in ASP.NET Core 2.1 or later.

## Configuration

In `ConfigureServices`, add the middleware to the service collection.

[!code-csharp[](middleware/samples/2.x/ResponseCachingMiddleware/Startup.cs?name=snippet1&highlight=9)]

Configure the app to use the middleware with the `UseResponseCaching` extension method, which adds the middleware to the request processing pipeline. The sample app adds a [`Cache-Control`](https://tools.ietf.org/html/rfc7234#section-5.2) header to the response that caches cacheable responses for up to 10 seconds. The sample sends a [`Vary`](https://tools.ietf.org/html/rfc7231#section-7.1.4) header to configure the middleware to serve a cached response only if the [`Accept-Encoding`](https://tools.ietf.org/html/rfc7231#section-5.3.4) header of subsequent requests matches that of the original request. In the code example that follows, [CacheControlHeaderValue](/dotnet/api/microsoft.net.http.headers.cachecontrolheadervalue) and [HeaderNames](/dotnet/api/microsoft.net.http.headers.headernames) require a `using` statement for the [Microsoft.Net.Http.Headers](/dotnet/api/microsoft.net.http.headers) namespace.

[!code-csharp[](middleware/samples/2.x/ResponseCachingMiddleware/Startup.cs?name=snippet2&highlight=17,21-28)]

Response Caching Middleware only caches server responses that result in a 200 (OK) status code. Any other responses, including [error pages](xref:fundamentals/error-handling), are ignored by the middleware.

> [!WARNING]
> Responses containing content for authenticated clients must be marked as not cacheable to prevent the middleware from storing and serving those responses. See [Conditions for caching](#conditions-for-caching) for details on how the middleware determines if a response is cacheable.

## Options

The middleware offers three options for controlling response caching.

| Option                | Description |
| --------------------- | ----------- |
| UseCaseSensitivePaths | Determines if responses are cached on case-sensitive paths. The default value is `false`. |
| MaximumBodySize       | The largest cacheable size for the response body in bytes. The default value is `64 * 1024 * 1024` (64 MB). |
| SizeLimit             | The size limit for the response cache middleware in bytes. The default value is `100 * 1024 * 1024` (100 MB). |

The following example configures the middleware to:

* Cache responses smaller than or equal to 1,024 bytes.
* Store the responses by case-sensitive paths (for example, `/page1` and `/Page1` are stored separately).

```csharp
services.AddResponseCaching(options =>
{
    options.UseCaseSensitivePaths = true;
    options.MaximumBodySize = 1024;
});
```

## VaryByQueryKeys

When using MVC/Web API controllers or Razor Pages page models, the `ResponseCache` attribute specifies the parameters necessary for setting the appropriate headers for response caching. The only parameter of the `ResponseCache` attribute that strictly requires the middleware is `VaryByQueryKeys`, which doesn't correspond to an actual HTTP header. For more information, see [ResponseCache Attribute](xref:performance/caching/response#responsecache-attribute).

When not using the `ResponseCache` attribute, response caching can be varied with the `VaryByQueryKeys` feature. Use the `ResponseCachingFeature` directly from the `IFeatureCollection` of the `HttpContext`:

```csharp
var responseCachingFeature = context.HttpContext.Features.Get<IResponseCachingFeature>();
if (responseCachingFeature != null)
{
    responseCachingFeature.VaryByQueryKeys = new[] { "MyKey" };
}
```

Using a single value equal to `*` in `VaryByQueryKeys` varies the cache by all request query parameters.

## HTTP headers used by Response Caching Middleware

Response caching by the middleware is configured using HTTP headers.

| Header | Details |
| ------ | ------- |
| Authorization | The response isn't cached if the header exists. |
| Cache-Control | The middleware only considers caching responses marked with the `public` cache directive. Control caching with the following parameters:<ul><li>max-age</li><li>max-stale&#8224;</li><li>min-fresh</li><li>must-revalidate</li><li>no-cache</li><li>no-store</li><li>only-if-cached</li><li>private</li><li>public</li><li>s-maxage</li><li>proxy-revalidate&#8225;</li></ul>&#8224;If no limit is specified to `max-stale`, the middleware takes no action.<br>&#8225;`proxy-revalidate` has the same effect as `must-revalidate`.<br><br>For more information, see [RFC 7231: Request Cache-Control Directives](https://tools.ietf.org/html/rfc7234#section-5.2.1). |
| Pragma | A `Pragma: no-cache` header in the request produces the same effect as `Cache-Control: no-cache`. This header is overridden by the relevant directives in the `Cache-Control` header, if present. Considered for backward compatibility with HTTP/1.0. |
| Set-Cookie | The response isn't cached if the header exists. Any middleware in the request processing pipeline that sets one or more cookies prevents the Response Caching Middleware from caching the response (for example, the [cookie-based TempData provider](xref:fundamentals/app-state#tempdata)).  |
| Vary | The `Vary` header is used to vary the cached response by another header. For example, cache responses by encoding by including the `Vary: Accept-Encoding` header, which caches responses for requests with headers `Accept-Encoding: gzip` and `Accept-Encoding: text/plain` separately. A response with a header value of `*` is never stored. |
| Expires | A response deemed stale by this header isn't stored or retrieved unless overridden by other `Cache-Control` headers. |
| If-None-Match | The full response is served from cache if the value isn't `*` and the `ETag` of the response doesn't match any of the values provided. Otherwise, a 304 (Not Modified) response is served. |
| If-Modified-Since | If the `If-None-Match` header isn't present, a full response is served from cache if the cached response date is newer than the value provided. Otherwise, a 304 (Not Modified) response is served. |
| Date | When serving from cache, the `Date` header is set by the middleware if it wasn't provided on the original response. |
| Content-Length | When serving from cache, the `Content-Length` header is set by the middleware if it wasn't provided on the original response. |
| Age | The `Age` header sent in the original response is ignored. The middleware computes a new value when serving a cached response. |

## Caching respects request Cache-Control directives

The middleware respects the rules of the [HTTP 1.1 Caching specification](https://tools.ietf.org/html/rfc7234#section-5.2). The rules require a cache to honor a valid `Cache-Control` header sent by the client. Under the specification, a client can make requests with a `no-cache` header value and force the server to generate a new response for every request. Currently, there's no developer control over this caching behavior when using the middleware because the middleware adheres to the official caching specification.

For more control over caching behavior, explore other caching features of ASP.NET Core. See the following topics:

* [Cache in-memory](xref:performance/caching/memory)
* [Work with a distributed cache](xref:performance/caching/distributed)
* [Cache Tag Helper in ASP.NET Core MVC](xref:mvc/views/tag-helpers/builtin-th/cache-tag-helper)
* [Distributed Cache Tag Helper](xref:mvc/views/tag-helpers/builtin-th/distributed-cache-tag-helper)

## Troubleshooting

If caching behavior isn't as expected, confirm that responses are cacheable and capable of being served from the cache. Examine the request's incoming headers and the response's outgoing headers. Enable [logging](xref:fundamentals/logging/index) to help with debugging.

When testing and troubleshooting caching behavior, a browser may set request headers that affect caching in undesirable ways. For example, a browser may set the `Cache-Control` header to `no-cache` or `max-age=0` when refreshing a page. The following tools can explicitly set request headers and are preferred for testing caching:

* [Fiddler](https://www.telerik.com/fiddler)
* [Postman](https://www.getpostman.com/)

### Conditions for caching

* The request must result in a server response with a 200 (OK) status code.
* The request method must be GET or HEAD.
* Terminal middleware, such as [Static File Middleware](xref:fundamentals/static-files), must not process the response prior to the Response Caching Middleware.
* The `Authorization` header must not be present.
* `Cache-Control` header parameters must be valid, and the response must be marked `public` and not marked `private`.
* The `Pragma: no-cache` header must not be present if the `Cache-Control` header isn't present, as the `Cache-Control` header overrides the `Pragma` header when present.
* The `Set-Cookie` header must not be present.
* `Vary` header parameters must be valid and not equal to `*`.
* The `Content-Length` header value (if set) must match the size of the response body.
* The [IHttpSendFileFeature](/dotnet/api/microsoft.aspnetcore.http.features.ihttpsendfilefeature) isn't used.
* The response must not be stale as specified by the `Expires` header and the `max-age` and `s-maxage` cache directives.
* Response buffering must be successful, and the size of the response must be smaller than the configured or default `SizeLimit`.
* The response must be cacheable according to the [RFC 7234](https://tools.ietf.org/html/rfc7234) specifications. For example, the `no-store` directive must not exist in request or response header fields. See *Section 3: Storing Responses in Caches* of [RFC 7234](https://tools.ietf.org/html/rfc7234) for details.

> [!NOTE]
> The Antiforgery system for generating secure tokens to prevent Cross-Site Request Forgery (CSRF) attacks sets the `Cache-Control` and `Pragma` headers to `no-cache` so that responses aren't cached. For information on how to disable antiforgery tokens for HTML form elements, see [ASP.NET Core antiforgery configuration](xref:security/anti-request-forgery#aspnet-core-antiforgery-configuration).

## Additional resources

* [Application Startup](xref:fundamentals/startup)
* [Middleware](xref:fundamentals/middleware/index)
* [Cache in-memory](xref:performance/caching/memory)
* [Work with a distributed cache](xref:performance/caching/distributed)
* [Detect changes with change tokens](xref:fundamentals/primitives/change-tokens)
* [Response caching](xref:performance/caching/response)
* [Cache Tag Helper](xref:mvc/views/tag-helpers/builtin-th/cache-tag-helper)
* [Distributed Cache Tag Helper](xref:mvc/views/tag-helpers/builtin-th/distributed-cache-tag-helper)
