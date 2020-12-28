# Django timescaledb

[![pypi-version]][pypi]


A database backend and tooling for Timescaledb.

Based on [gist](https://gist.github.com/dedsm/fc74f04eb70d78459ff0847ef16f2e7a) from WeRiot.

## Quick start

1. Install via pip

```bash
pip install django-timescaledb
```

2. Use as DATABASE engine in settings.py:

Standard PostgreSQL

```python
DATABASES = {
    'default': {
        'ENGINE': 'timescale.db.backends.postgresql',
        ...
    },
}
```

PostGIS

```python
DATABASES = {
    'default': {
        'ENGINE': 'timescale.db.backends.postgis',
        ...
    },
}
```

If you already make use of a custom PostgreSQL db backend you can set the path in settings.py.

```python
TIMESCALE_DB_BACKEND_BASE = "django.contrib.gis.db.backends.postgis"
```

3. Inherit from the TimescaleModel. A [hypertable](https://docs.timescale.com/latest/using-timescaledb/hypertables#react-docs) will automatically be created.

```python

  class TimescaleModel(models.Model):
    """
    A helper class for using Timescale within Django, has the TimescaleManager and 
    TimescaleDateTimeField already present. This is an abstract class it should 
    be inheritted by another class for use.
    """
    time = TimescaleDateTimeField(interval="1 day")

    objects = TimescaleManager()

    class Meta:
        abstract = True

```

Implementation would look like this

```python
from timescale.db.models.models import TimescaleModel

class Metric(TimescaleModel):
   temperature = models.FloatField()
   

```

If you already have a table and want to just add a field you can add the TimescaleDateTimeField to your model. This also triggers the creation of a hypertable.

```python
from timescale.db.models.fields import TimescaleDateTimeField
from timescale.db.models.managers import TimescaleManager

class Metric(models.Model):
  time = TimescaleDateTimeField(interval="1 day")

  objects = models.Manager()
  timescale = TimescaleManager()
```

The name of the field is important as Timescale specific feratures require this as a property of their functions.
### Reading Data

"TimescaleDB hypertables are designed to behave in the same manner as PostgreSQL database tables for reading data, using standard SQL commands."

As such the use of the Django's ORM is perfectally suited to this type of data. By leveraging a custom model manager and queryset we can extend the queryset methods to include Timescale functions.

#### Time Bucket [More Info](https://docs.timescale.com/latest/using-timescaledb/reading-data#time-bucket)

```python
  Metric.timescale.filter(time__range=date_range).time_bucket('time', '1 hour')

  # expected output

  <TimescaleQuerySet [{'bucket': datetime.datetime(2020, 12, 22, 11, 0, tzinfo=<UTC>)}, ... ]>
```

#### Time Bucket Gap Fill [More Info](https://docs.timescale.com/latest/using-timescaledb/reading-data#gap-filling)

```python
  from metrics.models import *
  from django.db.models import Count, Avg
  from django.utils import timezone
  from datetime import timedelta

  ranges = (timezone.now() - timedelta(days=2), timezone.now())

  (Metric.timescale
    .filter(time__range=ranges)
    .time_bucket_gapfill('time', '1 day', ranges[0], ranges[1])
    .annotate(Avg('temperature')))

  # expected output

  <TimescaleQuerySet [{'bucket': datetime.datetime(2020, 12, 21, 21, 24, tzinfo=<UTC>), 'temperature__avg': None}, ...]>
```

#### Histogram [More Info](https://docs.timescale.com/latest/using-timescaledb/reading-data#histogram)

```python
  from metrics.models import *
  from django.db.models import Count
  from django.utils import timezone
  from datetime import timedelta

  ranges = (timezone.now() - timedelta(days=3), timezone.now())

  (Metric.timescale
    .filter(time__range=ranges)
    .values('device')
    .histogram(field='temperature', min_value=50.0, max_value=55.0, num_of_buckets=10)
    .annotate(Count('device')))
    
  # expected output

  <TimescaleQuerySet [{'histogram': [0, 0, 0, 87, 93, 125, 99, 59, 0, 0, 0, 0], 'device__count': 463}]>
```