# Climbing the Test Pyramid

In this modern day of software development, we've learned that to deliver
high-quality software, we must work in an Agile way. To do this, we need quick
feedback so that we can adjust and learn as fast as possible. We
technically optimise the feedback cycle time at the smallest level by using
TDD, and at the largest level with Continuous Deployment.

We also know that to release fast and often, we need a test suite which gives
us complete confidence so that we don't release broken software.

With this knowledge, the problem which many people are struggling with at the
moment is slow test suites - which in turn, makes the build pipelines slow,
painful and expensive.

Thankfully there are a some very clever people who are currently sharing
information (at conferences and in books and articles) about building
architectures which allow this problem to be solved. This is done by favouring
isolated, targeted tests at lowest levels; over slow, all encompassing
end-to-end tests.

In a lot of people, this raises the concern how to maintain the
confidence that these lower level tests cover all the behaviour in the system,
although not testing completely everything with end-to-end acceptance tests.

For some years now, I've been developing software in this way, and I feel that
I have now learned a workflow which gives me complete confidence in the test
suite as well as fast build pipelines. This article presents the three simple
rules of this workflow.

[Jump straight to the rules](#the-rules)

## Introducing the Test Pyramid

The Test Pyramid was created quite some time ago by Mike Cohn. It looks like
this:

         /\
        /E2E      <- end-to-end
       /----\
      /Acceptance
     /--------\
    /___Unit___\

Each layer describes a level of testing in the project. The bottom level
contains the smallest, most targeted tests, which are the fastest to run. The
top level contains the tests which prove the system behaviour by testing the
whole system - these tests can be very slow.

To make the test suite fast, you want to create more tests in the lower levels,
which give the confidence to have less in the higher levels - how to do this is
the focus of this article.

It's worth noting some important properties of the test pyramid:

    slow   general expensive     /\
     |       |        |         /__\
     |       |        |        /____\
     |       |        |       /______\
    fast  focused   cheap    /________\
                                               |num. tests|

_**You can only have confidence in a layer if you have complete confidence in all
the layers below.**_

The names of the layers and the number of layers may vary depending on your
system architecture, but the properties of the pyramid and the rules I'm about
to present still apply.

It's also important to point out that the tests are run from the bottom of the
pyramid up. Each layer is only run if the layer below has passed successfully.

## The Rules

I'm going to introduce the rules now. The rest of this article demonstrates
how and why they work.

1. **Always start implementing a new system behaviour by writing a test at the
   lowest level where it can be described in a single test.**

2. **If a test is red, make it green by applying outside-in TDD down into the
   lower levels.**

3. **Only if all tests are green and the system behaviour is still not working,
   climb up to the next level of the pyramid and write a new failing test.**

The workflow for applying these rules looks like this:

![Workflow diagram](pyramid.jpg)

In my experience, these rules, used with a supporting architecture (like the
one described in [this
talk](https://skillsmatter.com/skillscasts/8567-testable-software-architecture)),
have always lead me to a fast and stable test suite which gives me the
confidence I need to run and change my application.

## The Example

For this example, we'll work through the process of creating a small system
which calculates wages from hours worked.

For this application:

* the UI will be a HTML webpage and will only be tested by the **E2E** tests.
* The business logic will all be contained in the *domain model*. This logic
  will be clearly defined by the tests in the **Acceptance** layer.
* The **Unit** layer will contain all the unit tests which are used to drive
  the implementation of the *domain model*.

### The First Behaviour

The first thing we need the system to do is to calculate the wages from the
number of hours worked. Let's create an example which describes this behaviour:

```gherkin
Scenario: Pay wages for the number of hours worked in a week
  Given Kevin gets paid £10 per hour
  And normal working hours are between 09:00 and 17:00
  And Kevin has worked for the following times:
    | Day        | Start | End   |
    | 2016-11-14 | 09:00 | 17:00 |
    | 2016-11-15 | 09:00 | 13:00 |
    | 2016-11-17 | 14:00 | 17:00 |
  When I pay Kevin's wages for the week starting on 2016-11-14
  Then Kevin should get £150 for the week starting on 2016-11-14
```

This is too complicated to create a single unit test for, but it can be
described in a single test at both the **E2E** or the **Acceptance** level. By
applying the **first rule**, we choose the lowest level of the two create a
test which looks like this:

```php
public function test_paying_wages_for_a_week()
    // Given Kevin gets paid £10 per hour
    $this->commandBus->run(new EmployeeDayRate($this->kevin, Money::fromPounds(10));

    // And normal working hours are between 09:00 and 17:00
    $this->commandBus->run(new SetWorkingHours(Period::fromStrings('09:00', '17:00')));

    // And Kevin has worked for the following times:
    //   | Day        | Start | End   |
    //   | 2016-11-14 | 09:00 | 17:00 |
    //   | 2016-11-15 | 09:00 | 13:00 |
    //   | 2016-11-17 | 14:00 | 17:00 |
    $this->commandBus->run(new LogWork($this->kevin, Date::fromString('2016-11-14'), Period::fromString('09:00', '17:00')));
    $this->commandBus->run(new LogWork($this->kevin, Date::fromString('2016-11-15'), Period::fromString('09:00', '13:00')));
    $this->commandBus->run(new LogWork($this->kevin, Date::fromString('2016-11-17'), Period::fromString('14:00', '17:00')));
   
    // When I calculate Kevin's wages for the week starting on 2016-11-14
    $this->commandBus->run(PayWeeklyWages($this->kevin, '2016-11-14');

    // Then Kevin should get £150 for the week starting on '2016-11-14'
    $result = $this->queryRunner(CheckEmployeeWagesForAWeek('2016-11-14'));
    assertEquals(150, $result->getAmount());
}
```

This test is now red because there is no implementation yet. To implement it we
can apply the **second rule** and use the TDD cycle to write unit tests and
drive the development of domain model.

Once all the tests are green, we can consider **rule three**. Since there is no
user interface at this point, we can consider system behaviour is incomplete.
Therefore, we can drive the creation of the user interface by **climbing up the
pyramid one level** to create an E2E test.

This E2E looks like this:

```php
public function test_paying_wages_for_a_week_via_the_UI()
    // Given Kevin gets paid £10 per hour
    $this->visit('/');
    $this->clickLink('Set Employee Rate');
    $this->clickLink('Kevin');
    $this->fillField('Rate', 10);
    $this->pressButton('Update');

    // And normal working hours are between 09:00 and 17:00
    $this->visit('/');
    $this->clickLink('Settings');
    $this->fillField('Start Time', '09:00');
    $this->fillField('End Time', '17:00');
    $this->pressButton('Update');

    // And Kevin has worked for the following times:
    //   | Day        | Start | End   |
    //   | 2016-11-14 | 09:00 | 17:00 |
    //   | 2016-11-15 | 09:00 | 13:00 |
    //   | 2016-11-17 | 14:00 | 17:00 |
    $this->visit('/');
    $this->clickLink('Set Employee Rate');
    $this->clickLink('Kevin');

    $this->fillField('Day', '2016-11-14');
    $this->fillField('Start', '09:00');
    $this->fillField('End', '17:00');
    $this->pressButton('Log Work');

    $this->fillField('Day', '2016-11-15');
    $this->fillField('Start', '09:00');
    $this->fillField('End', '13:00');
    $this->pressButton('Log Work');

    $this->fillField('Day', '2016-11-17');
    $this->fillField('Start', '14:00');
    $this->fillField('End', '17:00');
    $this->pressButton('Log Work');
   
    // Then Kevin should get £150 for the week starting on '2016-11-14'
    $this->visit('/');
    $this->clickLink('Calculate Wages for Week');
    $this->clickLink('Kevin');
    $this->fillField('week', '2016-11-14');
    $this->pressButton('Calculate');

    $this->assertPageHasText("Wages for week starting 2016-11-14: £150");
}
```

This test can now be used to drive the implementation of the user interface.

Once all the tests are green, we can manually check the behaviour of the system
and confirm that the implementation is complete.

### Second Behaviour

For the second behaviour, the system needs to pay the number of hours times one
and a half for overtime. Let's create a new scenario for that:

```gherkin
Scenario: Pay number of hours times one and a half for overtime
  Given Kevin gets paid £10 per hour
  And normal working hours are between 09:00 and 17:00
  And Kevin has worked for the following times:
    | Day        | Start | End   |
    | 2016-11-21 | 15:00 | 19:00 |
  When I pay Kevin's wages for the week starting on 2016-11-21
  Then Kevin should get £50 for the week starting on 2016-11-21
```

Again, we start by applying **rule one** - the lowest level at which this
scenario can be implemented is the **Acceptance** level, so we create a test
there:

```php
public function test_paying_overtime_wages_for_a_week()
    // Given Kevin gets paid £10 per hour
    $this->commandBus->run(new EmployeeDayRate($this->kevin, Money::fromPounds(10));

    // And normal working hours are between 09:00 and 17:00
    $this->commandBus->run(new SetWorkingHours(Period::fromStrings('09:00', '17:00')));

    // And Kevin has worked for the following times:
    //   | Day        | Start | End   |
    //   | 2016-11-21 | 15:00 | 19:00 |
    $this->commandBus->run(new LogWork($this->kevin, Date::fromString('2016-11-21'), Period::fromString('15:00', '19:00')));
   
    // When I calculate Kevin's wages for the week starting on 2016-11-21
    $this->commandBus->run(PayWeeklyWages($this->kevin, '2016-11-21');

    // Then Kevin should get £50 for the week starting on '2016-11-21'
    $result = $this->queryRunner(CheckEmployeeWagesForAWeek('2016-11-21'));
    assertEquals(50, $result->getAmount());
}
```

Next, this test is currently red, so we can apply **rule two** and push down
the pyramid layers to implement this using TDD. No changes to the UI are
required to do this.

Once the test goes green, we can apply **rule three** by going to the user
interface and checking if this behaviour works. If everything has gone well,
then this behaviour should be working correctly and there is **no need to
climb the pyramid and write another test**.

## Gherkin

At this point, all the business rules are tested and documented by the tests
in the **Acceptance** layer. This is the closest level to the logic where they can 
be described in business terms.

Since the test layers above the service layer are mostly made up subsets of the
layers below, it's nice to describe them in the same way. A neat way to do
this, which is slowly becoming more popular in the BDD community, is to use the
same Gherkin features at each level but have them test the application through
different entry points. This can either be done by running them against a
different set of *step definitions* or by injecting a different *driver* into
the test suite. Tags can be used to set which tests are promoted up to the
higher levels in the pyramid.

An example feature for the application we've just created would look like this:

```gherkin
Feature: Paying wages
  In order to pay our employees for the great work they do
  As a book keeper
  I need to be able to calcualte their wages

  Background:
    Given Kevin gets paid £10 per hour
    And normal working hours are between 09:00 and 17:00

  @e2e # <- promote this test to run at the E2E as well as the Service level
  Scenario: Pay wages for the number of hours worked in a week
    Given Kevin has worked for the following times:
      | Day        | Start | End   |
      | 2016-11-14 | 09:00 | 17:00 |
      | 2016-11-15 | 09:00 | 13:00 |
      | 2016-11-17 | 14:00 | 17:00 |
    When I pay Kevin's wages for the week starting on 2016-11-14
    Then Kevin should get £150 for the week starting on 2016-11-14

  Scenario: Pay number of hours times one and a half for overtime
    Given Kevin has worked for the following times:
      | Day        | Start | End   |
      | 2016-11-21 | 15:00 | 19:00 |
    When I pay Kevin's wages for the week starting on 2016-11-21
    Then Kevin should get £50 for the week starting on 2016-11-21
```

## Conclusion

From my current experience, these rules are a great guide to creating a test
suite which has a healthy shaped pyramid - giving high confidence and a fast
delivery. It doesn't matter what the layers of your test suite are, so long as
the lower ones support the ones above. For example, if your application has a
Javascript *Single Page Application* for the UI, which talks a REST API, then
the pyramid might look like this:

              /\
          End-to-End
            /Tests
           /-----\
          /API Tests
         /over HTTP\
        /-----------\
       / UI & Domain \
      /Acceptance Tests
     /----------------\
    /    UI & Domain   \
   /_____Unit Tests_____\

This article didn't cover everything which is necessary to achieve this - the
identification of the different layers, classification of the tests, correct
architecture and mocking approaches are all the things I didn't discuss in
detail here as I believe they have been covered in depth elsewhere.

Since I've come up with these three rules through my experience, I'd really
like to hear if they match your workflow or if you can think of situations
where they don't apply. If you have any further thoughts or questions please
contact me either by [email](mailto:tom@x2k.co.uk) or on
[Twitter](https://twitter.com/tomphp).
