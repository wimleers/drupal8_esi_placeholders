# ESI placeholder strategy for Drupal 8
This Drupal 8 module offers a placeholder strategy that renders [Edge Side Includes](https://en.wikipedia.org/wiki/Edge_Side_Includes).

## How does ESI work?

The Edge Side Include tags that are generated by this module serve as placeholders that are processed on *"the edge"*.

This is what an ESI tag could look like:

```<esi:include src="http://example.com/esi/block/custom-esi-block">```

A reverse caching proxy that supports *ESI* will process these tags and load the corresponding content that is referenced through the `src` attribute of the tag.

The benefit is that you can still serve pages from cache when certain blocks are uncacheable. The compositions of these placeholders into the final output doesn't happen in Drupal, but is done by the reverse caching proxy.  Hence the term *"on the edge"*.

## Which reverse caching proxies are supported?

A lot of *Content Delivery Networks* support ESI. The [ESI spec](https://www.w3.org/TR/esi-lang) was actually drafted in part by [https://www.akamai.com/](Akamai).

Here's a short list of CDN providers that comes to mind that support ESI:

* [Akamai](https://www.akamai.com/)
* [CloudFlare](https://www.cloudflare.com/)
* [Fastly](https://www.fastly.com/)

### Varnish

Even if you don't use a CDN, you can still use ESI. [Varnish](http://varnish-cache.org/) is a popular reverse caching proxy and is well-supported by Drupal.

If you want to add ESI support to your Varnish servers, just add the following snippet to to your VCL file.

```
sub vcl_recv {
    set req.http.Surrogate-Capability="key=ESI/1.0";
}

sub vcl_backend_response {
    if(beresp.http.Surrogate-Control ~ "ESI/1.0") {
        unset beresp.http.Surrogate-Control;
        set beresp.do_esi=true;
    }
}
```

## Inspired by BigPipe
The [BigPipe](https://www.drupal.org/project/big_pipe) module was my main source of inspiration. Some concepts were used to compose this render strategy.

The [EsiStrategy](/src/Render/Placeholder/EsiStrategy.php) class actually inherits from the `BigPipeStrategy` class, and the [generateBigPipePlaceholderId](/src/Render/Placeholder/EsiStrategy.php#L33) method is implemented in `BigPipeStrategy`.

The main difference is that the placeholders aren't parsed by Drupal, but by the reverse caching proxy. Another difference is a different markup style for the placeholders.

## How is the ESI placeholder strategy implemented?

The [auto-placeholdering](https://www.drupal.org/docs/8/api/render-api/auto-placeholdering) mechanism in Drupal 8 is responsible for turning blocks into placeholders.
The placeholdering strategies that are loaded, are responsible for turning these render arrays into something that optimizes the loading process.

### The EsiStrategy class

The [EsiStrategy](/src/Render/Placeholder/EsiStrategy.php) class depends on the `Request` and the `Esi` objects as dependencies. The [ESI](https://api.symfony.com/4.0/Symfony/Component/HttpKernel/HttpCache/Esi.html) object is part of the *Symfony HTTP kernel* and has a bunch of helper methods.
 
The `$this->esi->hasSurrogateCapability($request)` method call will check if the reverse caching proxies exposes the correct `Surrogate-Capability="key=ESI/1.0"` header.If that is the case and the individual placerholders contain a `#lazy_builder` key, ESI placeholders will be generated.

The `$this->esi->renderIncludeTag` method call will return an `<esi:include src="https://..." />` tag that points to an URL that contains the output for this block.

The render array is identified by an ID that is composed by the `generateBigPipePlaceholderId` method that comes directly from `BigPipe`.


```php
<?php
namespace Drupal\esi_placeholders\Render\Placeholder;

use Drupal\big_pipe\Render\Placeholder\BigPipeStrategy;
use Drupal\Core\Render\Markup;
use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\HttpKernel\HttpCache\Esi;
class EsiStrategy extends BigPipeStrategy
{
    /**
     * @var RequestStack
     */
    protected $requestStack;
    /**
     * @var Esi
     */
    protected $esi;

    /**
     * EsiStrategy constructor.
     * @param RequestStack $request_stack
     * @param Esi $esi
     */
    public function __construct(RequestStack $request_stack, Esi $esi)
    {
        $this->requestStack = $request_stack;
        $this->esi = $esi;
    }

    /**
     * @param array $placeholders
     * @return array
     */
    public function processPlaceholders(array $placeholders)
    {
        $request = $this->requestStack->getCurrentRequest();
        $overridenPlaceHolder = [];
        foreach ($placeholders as $placeholder => $placeholder_elements) {
            if (isset($placeholder_elements['#lazy_builder']) && $this->esi->hasSurrogateCapability($request)) {
                $overridenPlaceHolder[$placeholder] = [
                    '#markup' =>
                        Markup::create(
                            $this->esi->renderIncludeTag(
                                '/esi/block/?'.
                                $this->generateBigPipePlaceholderId($placeholder,$placeholder_elements),
                                null,
                                false
                        )
                    )
                ];
            }
        }
        return $overridenPlaceHolder;
    }
}
```

### The EsiController class

The [EsiController](/src/Controller/EsiController.php) class serves as the endpoint of the placeholder where the content is displayed.
A custom route is exposed in [esi_placeholders.routing.yml](/esi_placeholders.routing.yml), which points to `/esi/block/{blockId}`.

The block ID in the URL contains the `#lazybuilder` callback and the arguments for the callback, which makes it quite easy to display, as illustrated below:

```php
<?php
namespace Drupal\esi_placeholders\Controller;

use Drupal\Core\Controller\ControllerBase;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

class EsiController extends ControllerBase
{
    public function returnEsiBlockContent(Request $request)
    {
        $build = [
            'esiBlockContent' => [
                '#lazy_builder' => [
                    $request->get('callback'),
                    $request->get('args'),
                ]
            ]
        ];
        $output = \Drupal::service('renderer')->renderRoot($build);
        $response = new Response($output);
        return $response;
    }
}
```

By using the `renderRoot` function, only the HTML of the corresponding block is displayed, without including the theme.

### The EsiSubscriber class

The [EsiSubscriber](/src/EventSubscriber/EsiSubscriber.php) class is an event listener that adds the `Surrogate-Control: content="ESI/1.0"` response header if ESI support is detected and if the content contains ESI tags, as illustrated below:

```php
<?php

namespace Drupal\esi_placeholders\EventSubscriber;

use Symfony\Component\HttpKernel\Event\FilterResponseEvent;
use Symfony\Component\HttpKernel\HttpCache\Esi;
use Symfony\Component\HttpKernel\KernelEvents;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class EsiSubscriber implements EventSubscriberInterface
{
    /**
     * @var Symfony\Component\HttpKernel\HttpCache\Esi
     */
    protected $esi;

    /**
     * @param Symfony\Component\HttpKernel\HttpCache\Esi $esi
     */
    public function __construct(Esi $esi)
    {
        $this->esi = $esi;
    }

    /**
     * @param FilterResponseEvent $event
     * @return \Symfony\Component\HttpFoundation\Response
     */
    public function onRespond(FilterResponseEvent $event)
    {
        $request = $event->getRequest();
        $response = $event->getResponse();

        if($this->esi->hasSurrogateCapability($request)){
            $this->esi->addSurrogateControl($response);
            return $response;
        }
    }

    /**
     * {@inheritdoc}
     */
    public static function getSubscribedEvents()
    {
        $events[KernelEvents::RESPONSE][] = ['onRespond', -10000];
        return $events;
    }

}
```

## Why are these Surrogate headers required?

Performing ESI parsing and processing for every response, consumes quite a bit of server resources. 

In order to keep the process efficient, a 2-step handshake happens base on a `Surrogate-Capability` request header and a `Surrogate-Control` response header.

Your reverse caching proxy can announce ESI support by advertising this in a `Surrogate-Capability` header.

The following header is sent to Drupal by the proxy:

```
Surrogate-Capability: key="ESI/1.0"
```

The `$this->esi->hasSurrogateCapability($request)` method detects if this request header is sent.

Once Drupal decides to use ESI placeholders, it needs to announce this to the reverse caching proxy. It does this using the `Surrogate-Control` header:

```
Surrogate-Control: content="ESI/1.0"
``` 

Once your proxy receives this confirmation, it can go ahead and process the ESI tags. 

## Improvements

This is a pretty basic implementation without a lot of bells and whistles. The only use case it serves, is to display uncacheable content in placeholders of cacheable pages.

Offering placeholders with *TTLs* other than zero, is not yet supported.

I'm not sure if passing along the render array is a query string parameter, poses a security risk. There is a token that is passed, but I'm not sure how to validate it.

> Anyway, there's room for improvement, pull requests welcome!

