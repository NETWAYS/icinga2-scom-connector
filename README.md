SCOM Connector for Icinga 2
===========================

The SCOM connector will help you import alert data from SCOM into Icinga 2.

It helps you to update Icinga's configuration with Hosts and Services (not the
Icinga/Nagios meaning of service) from SCOM.

The other part of the tool will query all open alerts from SCOM and publish that
status into Icinga, so you can see and get alerted about open problems.

## Requirements

* Perl, at least 5.10 (tested on that platform)
* FreeTDS, the MSSQL database API for Unixoid systems
* DBD::Sybase for perl - the perl database driver to FreeTDS
* DBD::mysql for perl - for connecting to Icinga's IDO

You also need:
* Read-only access to SCOM's MSSQL database
* Read-only access to Icinga's IDO database

## How it works

Have a look at `icinga2_scom.conf`, it contains some basic configuration Icinga
must have for the integration. You can edit some of the objects, for e.g. to add
vars you need in your setup.

This basic file will give you a few things:
* CheckCommand - to run the state sync
* Host template
* Service template
* Host "SCOM Connector"
  * Service "SCOM Alert collection" - this runs the state synchronization
  * Service "SCOM unassigned Alerts" - alerts that can not be assigned get send here
* Host "SCOM Services"

When you run the tool in config mode:

```
scom_connector config
```

It will output:

* a Host object for every computer monitored by SCOM (dummy check from template)
* with a Service called "SCOM Alerts" (in passive mode)
* a Icinga Service for every SCOM Service monitored (assigned to the Host "SCOM Services")

TODO: example

The Service "SCOM Alert collection" from Host "SCOM Connector" will take care
about running status updated every minute.

```
scom_connector monitor
```

That will do the following:

1. Retrieve all alerts from SCOM's database
  * will be aggregated by host, server, and other alerts (generic SCOM alerts)
2. Get all SCOM hosts and services Icinga knows about
   (so we can avoid problems with config drift)
3. For each hosts and service build a summary of active alerts
4. Summarize all alerts that could not be assigned to an object Icinga knows
5. Submit the status to Icinga via its command pipe
6. Clear the status of objects we have not updated with alerts in this run
   (again via command pipe)

TODO: debug output as example

This means we should update the status of every SCOM object every time the tool
runs.

TODO: screenshots

## Install

Install all needed requirements.

Put the scom_connector somewhere Icinga can execute it, I'd recommend
`/etc/icinga2/plugins/scom_connector`.

Copy `scom_connector.conf.example` to `/etc/icinga2/plugins/scom_connector.conf`
and edit it to match your environment. The file can be put somewhere else when
you use an `-C` argument on execution.

Note: In addition you should make sure to set `tds version` in FreeTDS config,
usually at `/etc/freetds.conf`.

```
[global]
tds version = 7.2
```

Test if the configuration outputted makes sense:

```
/etc/icinga2/plugins/scom_connector config
```

The monitor command should publish everything under unassigned:

```
/etc/icinga2/plugins/scom_connector monitor
```

Now put all the config files in Icinga's config space:

```
mkdir /etc/icinga2/conf.d/scom
cp icinga2_scom.conf /etc/icinga2/conf.d/scom/basic.conf
/etc/icinga2/plugins/scom_connector config >/etc/icinga2/conf.d/scom/generated.conf
service icinga2 reload
```

When you run the tool in debug mode you should see what it does:
```
/etc/icinga2/plugins/scom_connector monitor -d
```

Make sure to update the config regularly, maybe by Cron.

## Configuration

The config file should be put in the same directory as the script, thats where
the script will look by default.

You can use any other path with adding `-C /fullpath/foobar.conf` on execution.

Any content should be valid Perl code, be careful.

By default you should only configure:
* SCOM database connection
* Icinga IDO connection
* Icinga's command pipe location

See `scom_connector.conf.example` for examples.

## License

This tool was created during my work as a consultant for [NETWAYS](http://www.netways.de) - we love open source.

Copyright (c) 2015 Markus Frosch <markus@lazyfrosch.de>, NETWAYS GmbH <info@netways.de>

This program is free software; you can redistribute it and/or modify it under
the terms of the GNU General Public License as published by the Free Software
Foundation; either version 2 of the License, or (at your option) any later
version.

This program is distributed in the hope that it will be useful, but WITHOUT
ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS
FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You can find a full version of the license in the LICENSE file.
