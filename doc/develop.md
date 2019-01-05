# CiviCRM Replay-on-Write: Development

## Installation (for local development)

If you have created a local D7 development site using `civibuild`, and if you've already
[configured Redis](https://docs.civicrm.org/sysadmin/en/latest/setup/cache/), then you can simulate
a master/slave toplogy using the script `rebuild-ro`.

```
## Get the code
cd /var/www/sites/default/civicrm/ext
git clone https://github.com/totten/rpow
cd rpow

## Setup a config file, esp:
## - CIVIRO_PASS (for security)
## - SITE_ROOT (for convenience)
cp etc/rebuild-ro.conf.example etc/rebuild-ro.conf

## Create a read-only DB. Register the DSN via civicrm.settings.d.
./bin/rebuild-ro
```

The `rebuild-ro` script will:

* Make a new database
* Copy the CiviCRM tables to the new database
* Add a user with read-only permission for the new database
* Create a file `civicrm.settings.d/pre.d/100-civirpow.php` 
  to call `rpow_init()` with the appropriate credentials
  for the `masters` and `slaves`.

This is handy for simulating master=>slave replication manually. It does
not require any special mysqld options, but it does assume that you have a
`civibuild`-based environment.

## Usage Example (Development)

Here are some example steps to see it working in development:

* In the browser
    * Navigate to your CiviCRM dev site
    * Do a search for all contacts
    * View a contact
    * Edit their name and save
    * Make a mental note of the CID. (Ex: `123`)
    * __Note__: The saved changes do not currently appear in the UI. Why?
      Because they were saved to the read-write master, but we're viewing data from the
      read-only slave.
* In the CLI
    * Lookup the contact record (ex: `123`) in both the master (`civi`) and slave (`civiro`) databases.
      You should see that the write went to the master (`civi`) but not the slave (`civiro`).
      ```
      SQL="select id, display_name from civicrm_contact where id = 123;"
      echo $SQL | amp sql -N civi ; echo; echo; echo $SQL | amp sql -N civiro
      ```
    * Update the slave.
      ```
      ./bin/rebuild-ro
      ```

TIP: When you are done doing development, delete the file
`civicrm.settings.d/pre.d/100-civirpow.php`.  This will put your dev site back
into a normal configuration with a single MySQL DSN.

## Unit Tests

Simply run `phpunit` without any arguments.

## TODO

Add integration tests covering DB_civirpow

Add sticky reconnect feature -- for (eg) 2 minutes after a write, all
subsequent connections should continue going to the read-write master.

debug toolbar is wonky, and it's hard to tell if it's ux or underlying
behavior. change ux to be a full-width bar at the bottom which displays
all available info. (instead of requiring extra clicks to drilldown)

optimistic-locking doesn't work -- it always reads the timestamp from rodb
before reconnecting to rwdb. any use-case that does optimistic-locking needs
a hint to force the reconnect beforehand.

packaging as a separate project makes it feel a bit sketchy to drop hints
into civicrm-core.  consider ways to deal with this (e.g.  package as part
of core s.t.  the hint notation is built-in; e.g.  figure out a way to make
the hint-notation abstract...  like with a listner/dispatcher pattern...
but tricky b/c DB and some caches come online during bootstrap, before we've
setup our regular dispatch subsystem)