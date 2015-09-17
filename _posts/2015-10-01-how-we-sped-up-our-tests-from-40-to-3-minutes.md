---
layout: post
title:  "How we sped up our API's unit tests, from 40 minutes to 2.5 minutes"
tags: tests unit-testing functional-testing symfony mongodb
permalink: /:year/:month/:day/:title
---

In a project I recently joined, we have an application built with Symfony and MongoDB. One of the original goals of the application was to "have a long-term relationship between the developers and codebase". Naturally, this means writing lots of unit and functional tests.

In the first year, this worked well, with a team of up to 5 developers at a time writing backend Symfony code and tests. Over that time though, the time to run tests grew from around 5 minutes to 20 minutes.

In the second year, as the number of lines of code grew, and the number of tests grew too, we started to hit growing pains. The time to run the Symfony tests hit 40 minutes -- and that's not even including our frontend tests, which were taking just as long. When tests take this amount of time, developers cut corners and don't like to write tests. Our continuous integration server (Jenkins) auto-scales on Amazon EC2 to launch machines as needed; this meant we were burning money for "number of builds per day * 40 minutes * cost per machine" -- that's a non-trivial amount.

With up to 7 developers at a time now working on the Symfony application, and developers losing lots of time waiting for tests to run, something needed to be done.

So what should be the first step you should take when "something" is slow and you need to optimize it? Profiling!

From the results of profiling with xdebug, it instantly became obvious where the problem was: our fixtures. See, we have a large amount of complex fixtures (pre-set data, like users, companies and so on), and at the start of each of the then-970 tests, the database was being wiped clean and the fixtures reloaded.

Based on xdebug's result, the first optimization was clear: stop reloading the fixtures 970 times in our PHPUnit `setUp` method! To do this, a few steps were needed:

* When the tests start to run (in `bin/run-tests`), reload the fixtures once to a test database (set in an environment variable TEST_DATABASE_NAME).
* Make a backup of the database (done in a new file: `bin/dump-test-database`).
* Run test 1.
* After test 1, restore the database backup (done in a new file: `bin/restore-test-database`).
* Run test 2.
* Restore the database again.
* And so on.

<pre>
# bin/run-tests
export TEST_DATABASE_NAME=$( cat app/config/parameters.yml | grep 'mongodb_db:' | awk '{ print $2 }' )_test
bin/dump-test-database
bin/phpunit -c app/ $*
</pre>

<pre>
# bin/dump-test-database
app/console doctrine:mongodb:fixtures:load --env test
app/console doctrine:mongodb:schema:update --env test
mongodump --db $TEST_DATABASE_NAME --out app/cache/test-db-backup > /dev/null
</pre>

<pre>
# bin/restore-test-database
mongorestore --db $TEST_DATABASE_NAME --drop app/cache/test-db-backup/$TEST_DATABASE_NAME 1> /dev/null 2> /dev/null
</pre>

<pre>
// BaseTest.php
abstract class BaseTest extends \PHPUnit_Framework_TestCase
{
    protected $kernel;

    protected $container;

    protected function setUp()
    {
        parent::setUp();

        $this->kernel = new \AppKernel('test', false);
        $this->kernel->boot();

        $this->container = $this->kernel->getContainer();

        // ...

        $this->loadFixtures();
    }

    private function loadFixtures()
    {
        `bin/restore-test-database`;
    }
}
</pre>

With this change alone, the time taken to run PHPUnit dropped from 40 minutes to 3 minutes. (That's on a Vagrant virtual machine allocated 4 GB of RAM and 4 CPUs.) That's a pretty crazy improvement.

But of course, more can be done. In most tests, the database doesn't actually change. It's just read from, not written to. So we can use a handy (but little-documented) MongoDB command, `db.runCommand({ dbhash : 1 })`, to get a hash of the database (done in `bin/get-test-database-hash`), and only restore the database from the backup if any changes have been made.

<pre>
# bin/run-tests
export TEST_DATABASE_NAME=$( cat app/config/parameters.yml | grep 'mongodb_db:' | awk '{ print $2 }' )_test
bin/dump-test-database

# This restore seems to help the tests have the same database hash much more
# often than without the restore. I don't know why it helps, but it does.
bin/restore-test-database

# This hash is used in `BaseTest::loadFixtures()` to check if the previous
# test has changed anything in the database.
export TEST_DATABASE_HASH=$( bin/get-test-database-hash )

bin/phpunit -c app/ $*
</pre>

<pre>
# bin/get-test-database-hash
mongo $TEST_DATABASE_NAME --quiet --eval 'db.runCommand({ dbhash : 1 }).md5'
</pre>

<pre>
// BaseTest.php
private function loadFixtures()
{
    // When `bin/run-tests` is run, the fixtures are loaded, then backed up
    // via `mongodump`. We now restore the fixtures via `mongorestore`, avoiding
    // having to reload the fixtures through PHP, Symfony, services, events,
    // etc., saving a huge amount of time.

    $databaseHash = trim(`bin/get-test-database-hash`);

    $reloadFixtures = !isset($_SERVER['redacted_app_name_test_database_hash'])
        || $_SERVER['redacted_app_name_test_database_hash'] !== $databaseHash;

    if ($reloadFixtures) {
        `bin/restore-test-database`;
    }
}
</pre>

With this, the time to run PHPUnit dropped even more, from 3 minutes to 2.5 minutes.

Another optimization made was to remove the downloading of a image (the application logo, used in a fixture, was downloaded for every test from the production site instead of from a local file). This also saved a few seconds, and made it easier to work without an internet connection.

Even now, more can be done: PHP 7 will make the tests run even faster (current public benchmarks say PHP 7 is about twice as fast as 5.6). Upgrading to MongoDB 3, from version 2.4, would probably also help.

To be able to run the number of tests we have (now over 1000) in a short time is extremely useful, and developers seem less likely to try skip writing tests.

In a future post, I'll write about how we sped up our end-to-end Protractor tests.
