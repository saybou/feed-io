# feed-io

[![SensioLabsInsight](https://insight.sensiolabs.com/projects/9cabcb4b-695e-43fa-8b83-a1f9ecefea88/mini.png)](https://insight.sensiolabs.com/projects/9cabcb4b-695e-43fa-8b83-a1f9ecefea88)
[![Latest Stable Version](https://poser.pugx.org/debril/feed-io/v/stable.png)](https://packagist.org/packages/debril/feed-io)
[![Build Status](https://secure.travis-ci.org/alexdebril/feed-io.png?branch=master)](http://travis-ci.org/alexdebril/feed-io)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/alexdebril/feed-io/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/alexdebril/feed-io/?branch=master)
[![Code Coverage](https://scrutinizer-ci.com/g/alexdebril/feed-io/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/alexdebril/feed-io/?branch=master)

[feed-io](https://github.com/alexdebril/feed-io) is a PHP library built to consume and serve news feeds. It features:

- JSONFeed / Atom / RSS read and write support
- a Command line interface to read feeds
- Multiple feeds reading at once through asynchronous requests
- PSR-7 Response generation with accurate cache headers
- HTTP Headers support when reading feeds in order to save network traffic
- Detection of the format (RSS / Atom) when reading feeds
- Enclosure support to handle external medias like audio content
- PSR compliant logging
- Content filtering to fetch only the newest items
- Malformed feeds auto correction
- DateTime detection and conversion
- A generic HTTP ClientInterface
- Guzzle Client integration

Keep informed about new releases and incoming features : http://debril.org/category/feed-io

# Installation

Use Composer to add feed-io into your project's requirements :

```sh
    composer require debril/feed-io
 ```

# Requirements

feed-io requires :

- php 7.1+
- psr/log 1.0
- guzzlehttp/guzzle 6.2+

it suggests :
- monolog/monolog 1.10+

Monolog is not the only library suitable to handle feed-io's logs, you can use any PSR/Log compliant library instead.

## Still on PHP 5 ?

No problem, you can still install feed-io 3.0. This version will be supported until the end of PHP 5.6 security fixes (31 december 2018).

## Why skipping PHP 7.0 ?

feed-io 4 requires PHP 7.1+ because return types cannot be nullable in PHP 7.0.

# Fetching the repository

Do this if you want to contribute (and you're welcome to do so):

```sh
    git clone https://github.com/alexdebril/feed-io.git

    cd feed-io/

    composer install
```

# Unit Testing

You can run the unit test suites using the following command in the library's source directory:

```sh

    ./vendor/bin/phpunit

```

Usage
=====

## CLI

Let's suppose you installed feed-io using Composer, you can use its command line client to read feeds from your terminal :

```shell
./vendor/bin/feedio read http://php.net/feed.atom
```

You can specify the number of items you want to read using the --count option. The instruction below will display the latest item :

```shell
./vendor/bin/feedio read -c 1 http://php.net/feed.atom
```

## reading

feed-io is designed to read feeds across the internet and to publish your own. Its main class is [FeedIo](https://github.com/alexdebril/feed-io/blob/master/src/FeedIo/FeedIo.php) :

```php

// create a simple FeedIo instance
$feedIo = \FeedIo\Factory::create()->getFeedIo();

// read a feed
$result = $feedIo->read($url);

// or read a feed since a certain date
$result = $feedIo->readSince($url, new \DateTime('-7 days'));

// get title
$feedTitle = $result->getFeed()->getTitle();

// iterate through items
foreach( $result->getFeed() as $item ) {
    echo $item->getTitle();
}

```

### Asynchronous reading of several feeds at once

Thanks to Guzzle, feed-io is able to fetch several feeds at once through asynchronous requests. If you're willing to get more information about the way it works, you can read [Guzzle's documentation](http://docs.guzzlephp.org/en/stable/quickstart.html#async-requests).

To read feeds using asynchronous requests with feed-io, you need to send a pool of `\FeedIo\Async\Request` objects to `\FeedIo\FeedIo::readAsync` and handle the result with a `\FeedIo\Async\CallbackInterface` of your own. You can also use `\FeedIo\Async\DefaultCallback` in order to test the feature.

Each `\FeedIo\Async\Request` is a request you want to perform, it embeds the feed's URL and optionnally a `\DateTime` to define the `modified-since` attribute of the request.

The `CallbackInterface` instance needs two methods :

```php

  /**
   * @param Result $result
   */
  public function process(Result $result) : void;

  /**
   * @param Request $request
   * @param \Exception $exception
   */
  public function handleError(Request $request, \Exception $exception) : void;

```

`process()` is called on successful reading and parsing to let you process the result. Otherwise `handleError()` will be triggered on faulty calls. Here is an example : [PDOCallback](examples/PDOCallback.php)

## formatting an object into a XML stream

```php

// build the feed
$feed = new FeedIo\Feed;
$feed->setTitle('...');

// convert it into Atom
$atomString = $feedIo->toAtom($feed);

// or ...
$atomString = $feedIo->format($feed, 'atom');

```

## building a feed including medias

```php
// build the feed
$feed = new FeedIo\Feed;
$feed->setTitle('...');

$item = $feed->newItem();

// build the media
$media = new \FeedIo\Feed\Item\Media
$media->setUrl('http://yourdomain.tld/medias/some-podcast.mp3');
$media->setType('audio/mpeg');

// add it to the item
$item->addMedia($media);

$feed->add($item);

```

## Creating a valid PSR-7 Response with a feed

You can turn a `\FeedIo\FeedInstance` directly into a PSR-7 valid response using `\FeedIo\FeedIo::getPsrResponse()` :

```php

$feed = new \FeedIo\Feed;

// feed the beast ...
$item = new \FeedIo\Feed\Item;
$item->set ...
$feed->add($item);

$atomResponse = $feedIo->getPsrResponse($feed, 'atom');

$jsonResponse = $feedIo->getPsrResponse($feed, 'json');

```

## activate logging

feed-io natively supports PSR-3 logging, you can activate it by choosing a 'builder' in the factory :

```php

$feedIo = \FeedIo\Factory::create(['builder' => 'monolog'])->getFeedIo();

```

feed-io only provides a builder to create Monolog\Logger instances. You can write your own, as long as the Builder implements BuilderInterface.

## Building a FeedIo instance without the factory

To create a new FeedIo instance you only need to inject two dependencies :

 - an HTTP Client implementing FeedIo\Adapter\ClientInterface. It can be wrapper for an external library like FeedIo\Adapter\Guzzle\Client
- a PSR-3 logger implementing Psr\Log\LoggerInterface

```php

// first dependency : the HTTP client
// here we use Guzzle as a dependency for the client
$guzzle = new GuzzleHttp\Client();
// Guzzle is wrapped in this adapter which is a FeedIo\Adapter\ClientInterface  implementation
$client = new FeedIo\Adapter\Guzzle\Client($guzzle);

// second dependency : a PSR-3 logger
$logger = new Psr\Log\NullLogger();

// now create FeedIo's instance
$feedIo = new FeedIo\FeedIo($client, $logger);

```

Another example with Monolog configured to write on the standard output :

```php
use FeedIo\FeedIo;
use FeedIo\Adapter\Guzzle\Client;
use GuzzleHttp\Client as GuzzleClient;
use Monolog\Logger;
use Monolog\Handler\StreamHandler;

$client = new Client(new GuzzleClient());
$logger = new Logger('default', [new StreamHandler('php://stdout')]);

$feedIo = new FeedIo($client, $logger);

```

## Dealing with missing timezones

Sometimes you have to consume feeds in which the timezone is missing from the dates. In some use-cases, you may need to specify the feed's timezone to get an accurate value, so feed-io offers a workaround for that :

```php
$feedIo->getDateTimeBuilder()->setFeedTimezone(new \DateTimeZone($feedTimezone));
$result = $feedIo->read($feedUrl);
$feedIo->getDateTimeBuilder()->resetFeedTimezone();
```

Don't forget to reset `feedTimezone` after fetching the result, or you'll end up with all feeds located in the same timezone.

## Online documentation and API reference

The whole documentation and API reference is available at https://feed-io.net/documentation.html
