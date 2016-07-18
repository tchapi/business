# Business [![Build status][travis-image]][travis-url] [![Version][version-image]][version-url] [![PHP Version][php-version-image]][php-version-url]

> DateTime calculations in business hours

## Installation

```bash
$ composer require florianv/business
```

## Usage

First you need to configure your business schedule:

```php
use Business\SpecialDay;
use Business\Day;
use Business\Days;
use Business\Business;
use Business\Holidays;
use Business\DateRange;

// Opening hours for each week day. If not specified, it is considered closed
$days = [
    // Standard days with fixed opening hours
    new Day(Days::MONDAY, [['09:00', '13:00'], ['2pm', '5 PM']]),
    new Day(Days::TUESDAY, [['9 AM', '5 PM']]),
    new Day(Days::WEDNESDAY, [['10:00', '13:00'], ['14:00', '17:00']]),
    new Day(Days::THURSDAY, [['10 AM', '5 PM']]),
    
    // Special day with dynamic opening hours depending on the date
    new SpecialDay(Days::FRIDAY, function (\DateTime $date) {
        if ('2015-05-29' === $date->format('Y-m-d')) {
            return [['9 AM', '12:00']];
        }
    
        return [['9 AM', '5 PM']];
    }),
];

// Optional holiday dates
$holidays = new Holidays([
    new \DateTime('2015-01-01'),
    new \DateTime('2015-01-02'),
    new DateRange(new \DateTime('2015-07-08'), new \DateTime('2015-07-11')),
]);

// Optional business timezone
$timezone = new \DateTimeZone('Europe/Paris');

// Create a new Business instance
$business = new Business($days, $holidays, $timezone);
```

### Methods

##### within() - Tells if a date is within business hours

```php
$bool = $business->within(new \DateTime('2015-05-11 10:00'));
```

##### timeline() - Returns a timeline of business dates

```php
$start = new \DateTime('2015-05-11 10:00');
$end = new \DateTime('2015-05-14 10:00');
$interval = new \DateInterval('P1D');

$dates = $business->timeline($start, $end, $interval);
```

##### closest() - Returns the closest business date from a given date

```php
// After that date (including it)
$nextDate = $business->closest(new \DateTime('2015-05-11 10:00'));

// Before that date (including it)
$lastDate = $business->closest(new \DateTime('2015-05-11 10:00'), Business::CLOSEST_LAST);
```

### Symfony 2 & 3 Forms

If an entity depends on a field of type `Business`, you can easily create a specific type to use in your form builder :

```php
namespace Your\Form\Type;

use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\Component\Form\Extension\Core\Type\FormType;
use Symfony\Component\Form\Extension\Core\Type\ChoiceType;
use Symfony\Component\Form\Extension\Core\Type\CollectionType;
use Symfony\Component\Form\CallbackTransformer;
use Business\Days;
use Business\TimeInterval;

class DayType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder->add('dayOfWeek', ChoiceType::class, ['choices' => Days::toFormArray(), 'choices_as_values' => true]);
        $builder->add('openingIntervals', CollectionType::class, [
                'entry_type' => 'text',
                'allow_add' => true,
                'allow_delete' => true,
                'by_reference' => false
            ]);

        $builder->get('openingIntervals')
            ->addModelTransformer(new CallbackTransformer(
                function ($timeIntervalsAsArrayOfObjects) {
                    if ($timeIntervalsAsArrayOfObjects == null) {
                        return [];
                    }
                    $timeIntervalAsArrayOfStrings = [];
                    foreach ($timeIntervalsAsArrayOfObjects as $key => $value) {
                        $timeIntervalAsArrayOfStrings[$key] = (string) $value;
                    }
                    return $timeIntervalAsArrayOfStrings;
                },
                function ($timeIntervalAsArrayOfStrings) {
                    if ($timeIntervalAsArrayOfStrings == null || $timeIntervalAsArrayOfStrings == []) {
                        return [];
                    }
                    $timeIntervalsAsArrayOfObjects = [];
                    foreach ($timeIntervalAsArrayOfStrings as $key => $value) {
                        if ($value != "") {
                            $timeIntervalsAsArrayOfObjects[$key] = explode(TimeInterval::SEPARATOR, $value);
                        }
                    }
                    return $timeIntervalsAsArrayOfObjects;
                }
            ));
    }

    public function configureOptions(OptionsResolver $resolver)
    {
        $resolver->setDefaults(array(
            'data_class' => 'Business\Day',
        ));
    }

    public function getParent()
    {
        return FormType::class;
    }
}
```

And : 


```php
namespace Your\Form\Type;

use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\Component\Form\Extension\Core\Type\FormType;
use Symfony\Component\Form\Extension\Core\Type\TimezoneType;
use Symfony\Component\Form\Extension\Core\Type\CollectionType;
use Symfony\Component\Form\CallbackTransformer;

class BusinessType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder->add('days', CollectionType::class, [
                'entry_type' => DayType::class,
                'allow_add' => true,
                'allow_delete' => true,
                'by_reference' => false
            ]);

        $builder->add('holidays', CollectionType::class, [
                'entry_type' => 'datetime',
                'allow_add' => true,
                'allow_delete' => true,
                'by_reference' => false
            ]);
        $builder->add('timezone', TimezoneType::class);

        $builder->get('timezone')
            ->addModelTransformer(new CallbackTransformer(
                function ($timeZoneObject) {
                    return $timeZoneObject->getName();
                },
                function ($timeZoneString) {
                    return new \DateTimeZone($timeZoneString);
                }
            ));
    }

    public function configureOptions(OptionsResolver $resolver)
    {
        $resolver->setDefaults(array(
            'data_class' => 'Business\Business',
        ));
    }

    public function getParent()
    {
        return FormType::class;
    }
}
```


### Serialization

#### PHP serialization

The `Business` class can be serialized so it can be stored for later reuse:

```php
$serialized = serialize($business);
$business = unserialize($serialized);
```

If you use `SpecialDay` instances, you need to install the `jeremeamia/superclosure` library providing closure serialization:

```bash
$ composer require jeremeamia/superclosure
```

#### JSON serialization

All classes can be encoded to JSON

```php
$json = json_encode($business);
```

The `SpecialDay` instances require a context in order extract the data from the callable.
This is automatically set to `new \DateTime('now')` for `json_encode()` call only.

## License

[MIT](https://github.com/florianv/business/blob/master/LICENSE)

[travis-url]: https://travis-ci.org/florianv/business
[travis-image]: http://img.shields.io/travis/florianv/business.svg?style=flat

[version-url]: https://packagist.org/packages/florianv/business
[version-image]: http://img.shields.io/packagist/v/florianv/business.svg?style=flat

[php-version-url]: https://packagist.org/packages/florianv/business
[php-version-image]: http://img.shields.io/badge/php-5.4+-ff69b4.svg
