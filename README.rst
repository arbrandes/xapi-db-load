Scripts for generating Aspects xAPI events
******************************************

Purpose
=======
This package generates a variety of test data used for integration and
performance testing of Open edX Aspects. Currently it populates the following
datasets:

- xAPI statements, simulating those generated by event-routing-backends
- Course and learner data, simulating that generated by event-sink-clickhouse

The xAPI events generated match the current specifications of the Open edX
event-routing-backends package, but are not yet maintained to advance alongside
them so may be expected to fall out of sync over time. Almost all statements
supported by event-routing-backends are supported, except those not yet used in
Aspects reporting.

Features
========
New! UI mode is available for early testing using:

``xapi-db-load ui --config_file ...``.

Please add any issues you find here: https://github.com/openedx/xapi-db-load/issues

Data can be generated using the following backends:

clickhouse
----------
This backend issues batched insert statements directly against the configured
ClickHouse database. It is useful for getting a small to medium amount of data
(up 10s of thousands of xAPI events) into the database to test configuration and
view populated reports.


ralph
-----
This backend uses the configured Ralph Learning Record Store to handle the
inserting of xAPI events into ClickHouse. All other data is handled using the
clickhouse backend. It is useful for testing Ralph configuration, integration,
and permissions. This is the slowest method, but exercises the largest
surface area of the Aspects project.

csv
---
This backend generates a single gzipped CSV file for each type of data and
can save the files to local or various cloud storages. When saved to S3 it can
also load the files to ClickHouse immediately after creation, or at a later
time using the ``--load_db_only`` option.

Useful for creating datasets that can be reused for checking performance
changes with the exact same data, and for large scale tests. This is the
fastest method for medium to large scale tests (10K - 100M xAPI statements),
but can encounter "too many parts" errors when executing very large loads
(hundreds of millions
of xAPI statements) or loads that cover a period over 9 years between the
configured `start_date` and `end_date`.

chdb
----
This backend generates lz4 compressed ClickHouse Native files in S3 using the
CHDB in-process ClickHouse engine and can optionally load the files to
a ClickHouse service directly after creation or at a later time using the
``--load_db_only`` option. The generated files are partitioned differently per data
type to parallelize data writing and loading. This is the fastest engine for
very large scale tests and should be able to generate several billions of rows
of data across any length of time without error.



Getting Started
===============

Usage
-----

A configuration file is required to run a test. If no file is given, a small
test will be run using the `default_config.yaml` included in the project:

::

    ❯ xapi-db-load load-db

To specify a config file:

::

    ❯ xapi-db-load load-db --config_file private_configs/my_huge_test.yaml

There is also an option for just performing a load of previously generated
CSV data:

::

    ❯ xapi-db-load load-db --load_db_only --config_file private_configs/my_s3_test.yaml

To try out the new UI mode:

::

    ❯ xapi-db-load ui --config_file private_configs/my_huge_test.yaml



Configuration Format
--------------------
There are a number of different configuration options for tuning the output.
In addition to the documentation below, there are example settings files to
review in the ``example_configs`` directory.

Common Settings
^^^^^^^^^^^^^^^
These settings apply to all backends, and determine the size and makeup of the
test::

    # Location where timing logs will be saved
    log_dir: logs

    # xAPI statements will be generated in batches, the total number of
    # statements is ``num_xapi_batches * batch_size``. The batch size is the number
    # of xAPI statements sent to the backend (Ralph POST, ClickHouse insert, etc.)
    #
    # In the chdb backend "batch_size" is also used for any non-xAPI data that
    # uses batching inserts.
    num_xapi_batches: 3
    batch_size: 100

    # Overall start and end date for the entire run. All xAPI statements
    # will fall within these dates. Different courses will have different start
    # and end dates between these days, based on course_length_days below.
    start_date: 2014-01-01
    end_date: 2023-11-27

    # All courses will be this long, they will be fit between start_date and
    # end_date, therefore this must be less than end_date - start_date days.
    course_length_days: 120

    # The number of organizations, courses will be evenly spread among these
    num_organizations: 3

    # The number of learners to create, random subsets of these will be
    # "registered" for each course and have statements generated for them
    # between their registration date and the end of the course
    num_actors: 10

    # How many of each size course to create. The sum of these is the total
    # number of courses created for the test. The keys are arbitrary, you can
    # name them whatever you like and have as many or few sizes as you like.
    # The keys must exactly match the definitions in course_size_makeup below.
    num_course_sizes:
      small: 1
      medium: 1
      ...

    # Course type configurations, how many of each type of object are created
    # for each course of this size. "actors" must be less than or equal to
    # "num_actors". Keys here must exactly match the keys in num_course_sizes.
    course_size_makeup:
      small:
        actors: 5
        problems: 20
        videos: 10
        chapters: 3
        sequences: 10
        verticals: 20
        forum_posts: 20
      medium:
        actors: 7
        problems: 40
        videos: 20
        chapters: 4
        sequences: 20
        verticals: 30
        forum_posts: 40
      ...

ClickHouse Backend
^^^^^^^^^^^^^^^^^^
Backend is only necessary if you are writing directly to ClickHouse, for
integrations with Ralph or CSV, use their backend instead::

    backend: clickhouse

Variables necessary to connect to ClickHouse, whether directly, through Ralph, or
as part of loading CSV files::

    # ClickHouse connection variables
    db_host: localhost
    # db_port is also used to determine the "secure" parameter. If the port
    # ends in 443 or 440, the "secure" flag will be set on the connection.
    db_port: 8443
    db_username: ch_admin
    db_password: secret

    # Schema name for the xAPI schema
    db_name: xapi

    # Schema name for the event sink schema
    db_event_sink_name: event_sink

    # These S3 settings are shared with the CSV backend, but passed to
    # ClickHouse when loading files from S3
    s3_key: <...>
    s3_secret: <...>

Ralph / ClickHouse Backend
^^^^^^^^^^^^^^^^^^^^^^^^^^
Variables necessary to send xAPI statements via Ralph::

    backend: ralph_clickhouse
    lrs_url: http://ralph.tutor-nightly-local.orb.local/xAPI/statements
    lrs_username: ralph
    lrs_password: secret

    # This also requires all of the ClickHouse backend variables!


CSV Backend, Local Files
^^^^^^^^^^^^^^^^^^^^^^^^
Generates gzipped CSV files to a local directory::

    backend: csv_file
    csv_output_destination: logs/

CSV Backend, S3 Compatible Destination
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Generates gzipped CSV files to remote location::

    backend: csv_file
    # This can be anything smart-open can handle (ex. a local directory or
    # an S3 bucket etc.) but importing to ClickHouse using this tool only
    # supports S3 or compatible services like MinIO right now.
    # Note that this *must* be an s3:// link, https links will not work
    # https://pypi.org/project/smart-open/
    csv_output_destination: s3://openedx-aspects-loadtest/logs/large_test/

    # These settings are shared with the ClickHouse backend
    s3_key:
    s3_secret:

CSV Backend, S3 Compatible Destination, Load to ClickHouse
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Generates gzipped CSV files to a remote location, then automatically loads
them to ClickHouse::

    backend: csv_file
    # csv_output_destination can be anything smart_open can handle, a local
    # directory or an S3 bucket etc., but importing to ClickHouse using this
    # tool only supports S3 or compatible services (ex: MinIO) right now
    # https://pypi.org/project/smart-open/
    csv_output_destination: s3://openedx-aspects-loadtest/logs/large_test/
    csv_load_from_s3_after: true

    # Note that this *must* be an https link, s3:// links will not work,
    # this must point to the same location as csv_output_destination.
    s3_source_location: https://openedx-aspects-loadtest.s3.amazonaws.com/logs/large_test/

    # This also requires all of the ClickHouse backend variables above!

CHDB Backend, S3 Compatible Destination
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Generates lz4 compressed ClickHouse Native formatted files on S3::

    backend: chdb

    # This S3 configuration would upload files to s3://foo/logs/async_test using
    # the provided S3 key and secret
    s3_bucket: foo
    s3_prefix: logs/large_test/
    s3_key: ...
    s3_secret: ...


CHDB Backend, S3 Compatible Destination, Load to ClickHouse
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Generates lz4 compressed ClickHouse Native formatted files on S3, then
automatically load them to ClickHouse::

    backend: chdb

    # This S3 configuration would upload files to s3://foo/logs/async_test using
    # the provided S3 key and secret
    s3_bucket: foo
    s3_prefix: logs/large_test/
    s3_key: ...
    s3_secret: ...

    # After generating the files, load them from S3. With this backend, files
    # are loaded concurrently in the background while others are still being
    # generated.
    load_from_s3_after: true

    # This also requires all of the ClickHouse backend variables above!


Developing
----------

One Time Setup
^^^^^^^^^^^^^^

.. code-block::

  # Clone the repository
  git clone git@github.com:openedx/xapi-db-load.git
  cd xapi-db-load

  # Set up a virtualenv using virtualenvwrapper with the same name as the repo
  # and activate it
  mkvirtualenv -p python3.11 xapi-db-load


Every time you develop something in this repo
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block::

  # Activate the virtualenv
  workon xapi-db-load

  # Grab the latest code
  git checkout main
  git pull

  # Install/update the dev requirements
  make requirements

  # Run the tests and quality checks (to verify the status before you make any
  # changes)
  make validate

  # Make a new branch for your changes
  git checkout -b <your_github_username>/<short_description>

  # Using your favorite editor, edit the code to make your change.
  vim ...

  # Run your new tests
  pytest ./path/to/new/tests

  # Run all the tests and quality checks
  make validate

  # Commit all your changes
  git commit ...
  git push

  # Open a PR and ask for review.


Getting Help
============

Documentation
-------------

Start by going through `the documentation`_ (in progress!).

.. _the documentation: https://docs.openedx.org/projects/xapi-db-load


More Help
---------

If you're having trouble, we have discussion forums at
https://discuss.openedx.org where you can connect with others in the
community.

Our real-time conversations are on Slack. You can request a `Slack
invitation`_, then join our `community Slack workspace`_.

For anything non-trivial, the best path is to open an issue in this
repository with as many details about the issue you are facing as you
can provide.

https://github.com/openedx/xapi-db-load/issues

For more information about these options, see the `Getting Help`_ page.

.. _Slack invitation: https://openedx.org/slack
.. _community Slack workspace: https://openedx.slack.com/
.. _Getting Help: https://openedx.org/getting-help

License
*******

The code in this repository is licensed under the AGPL 3.0 unless
otherwise noted.

Please see `LICENSE.txt <LICENSE.txt>`_ for details.

Contributing
************

Contributions are very welcome.
Please read `How To Contribute <https://openedx.org/r/how-to-contribute>`_ for
details.

This project is currently accepting all types of contributions, bug fixes,
security fixes, maintenance work, or new features.  However, please make sure
to have a discussion about your new feature idea with the maintainers prior to
beginning development to maximize the chances of your change being accepted.
You can start a conversation by creating a new issue on this repo summarizing
your idea.

The Open edX Code of Conduct
****************************

All community members are expected to follow the `Open edX Code of Conduct`_.

.. _Open edX Code of Conduct: https://openedx.org/code-of-conduct/

People
******

The assigned maintainers for this component and other project details may be
found in `Backstage`_. Backstage pulls this data from the ``catalog-info.yaml``
file in this repo.

.. _Backstage: https://open-edx-backstage.herokuapp.com/catalog/default/component/xapi-db-load

Reporting Security Issues
*************************

Please do not report security issues in public. Please email security@openedx.org.
