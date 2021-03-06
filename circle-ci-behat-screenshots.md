# Behat on CircleCI with Failure Screenshots

***TIP
**tl;dr** By combining Behat and CircleCI, you can automatically take screenshots
on failures and see a list of them on each build. This is madness!
***

For KnpUniversity, we use both [Behat](https://knpuniversity.com/screencast/behat)
and CircleCI for continuous integration. The combination works *great*, except for
debugging. If you've done functional testing in a CI environment before, then
you're probably familiar with (yikes!) *phantom failures*: those tests that *only*
seem to fail on the CI server, making them nearly impossible to debug.

I don't like to lose time debugging tests. And thanks to Behat and CircleCI, you
can easily take screenshots of the browser at the moment of those failures and see
them on the build afterwards. Now, when you forget to install `wkhtmltopdf` on your
CI server, you'll see a big error screenshot that tells you so. Sweet!

## CircleCI Setup

To run our `@javascript` Behat scenarios, we use Selenium2. Our `circle.yml` setup
(without the screenshot magic) looks like this:

```yml
# circle.yml
machine:
  php:
    version: 5.5.21

dependencies:
  cache_directories:
    - vendor

  pre:
    - sudo composer self-update
    - composer install --prefer-dist

    # get selenium rocking!
    - wget http://selenium-release.storage.googleapis.com/2.47/selenium-server-standalone-2.47.1.jar
    - 'java -jar selenium-server-standalone-2.47.1.jar > /dev/null 2>&1':
          background: true

    # start the web server
    - 'php app/console server:run --env=test -vvv localhost:8080 > server.log 2>&1':
          background: true

database:
  override:
    - app/console do:da:cr -e=test
    - app/console do:sc:cr -e=test

test:
  override:
    - php vendor/bin/behat
    # and maybe some unit tests
```

Actually, CircleCI already takes care of a lot of details that we [previously](https://knpuniversity.com/screencast/question-answer-day/travis-ci)
needed to worry about, like setting up xvfb and installing a browser (chrome).

The `behat.yml` file doesn't need anything special, except that the web server port
needs to match the `8080` used above you should use the `chrome` browser:

```yml
default:
    # ...
    extensions:
        Behat\MinkExtension:
            base_url: http://localhost:8080
            goutte: ~
            selenium2: ~
            browser_name: chrome
        Behat\Symfony2Extension: ~
```

That's it for CircleCI setup.

## Taking Screenshots on Failure

Next, we need to take a screenshot each time a scenario fails. Behat does most of
the work for you. Just add this to your `FeatureContext`:

```php
// FeatureContext.php
use Behat\Behat\Hook\Scope\AfterStepScope;
use Behat\Mink\Driver\Selenium2Driver;
use Behat\MinkExtension\Context\RawMinkContext

// ...

class FeatureContext extends RawMinkContext
{
    /**
     * @AfterStep
     */
    public function printLastResponseOnError(AfterStepScope $event)
    {
        if (!$event->getTestResult()->isPassed()) {
            $this->saveDebugScreenshot();
        }
    }

    /**
     * @Then /^save screenshot$/
     */
    public function saveDebugScreenshot()
    {
        $driver = $this->getSession()->getDriver();

        if (!$driver instanceof Selenium2Driver) {
            return;
        }

        if (!getenv('BEHAT_SCREENSHOTS')) {
            return;
        }

        $filename = microtime(true).'.png';
        $path = $this->getContainer()
            ->getParameter('kernel.root_dir').'/../behat_screenshots';

        if (!file_exists($path)) {
            mkdir($path);
        }

        $this->saveScreenshot($filename, $path);
    }
}
```

This uses the [@AfterStep hook](http://knpuniversity.com/screencast/behat/behat-hooks-background)
to save a timestamped screenshot into a `behat_screenshots/` directory on failure.
The `saveScreenshot()` method comes directly from the `RawMinkContext` class that's
available from [MinkExtension](http://knpuniversity.com/screencast/behat/behat-loves-mink).

***TIP
With a little more work, you could probably use the `AfterStepScope` object to save
a screenshot that's derived from the scenario's name.
***

It also *only* does this if it sees a `BEHAT_SCREENSHOTS` environmental variable:
I don't need screenshots when I'm working locally.

To finish things, we need to tweak the CircleCI setup to make sure this directory
exists and to set the `BEHAT_SCREENSHOTS` environment variable that activates everything:

```yml
test:
  pre:
    - mkdir behat_screenshots
    - chmod 777 behat_screenshots
    - export BEHAT_SCREENSHOTS="1"
  override:
    - php vendor/bin/behat
    # and maybe some unit tests
  post:
    # add a file so that the directory isn't empty (else copy will fail)
    - touch behat_screenshots/keep
    - cp behat_screenshots/* $CIRCLE_ARTIFACTS/
    - rm $CIRCLE_ARTIFACTS/keep
```

The `$CIRCLE_ARTIFACTS` is an environment variable that's special to CircleCI: it
points to a directory where you can place artifacts. Anything you put here is automatically
made available in the artifacts section of your build after it finishes.

To become a Behat & Mink expert, check out our full
[BDD, Behat, Mink and other Wonderful Things](https://knpuniversity.com/screencast/behat)
tutorial.

Have fun!
