# driving-directions
Driving Directions for pgRouting

This project was started in 2008 to explicate driving directions for routes genereated using pgRouting. It has some data specific assumption in the code, but the basic structure should be easy to adapt to other data sets.

## Installation

The process of setting up a database involves:

1. load the street data
2. build the topology for pgRouting
3. load the driving direction code

And can be seen here by this stript, which probably will not work for your data unless you make adjustments to it.

```
#!/bin/sh

if [[ $1 == '' ]] ; then
    echo 'Usage: load-and-prep-data.sh dbname'
    echo 'NOTE: this will DROP DATABASE dbname if it exists!!!!'
    exit
fi

DBNAME=$1
DBUSER=postgres
DBPORT=5432
DBHOST=localhost

echo "Drop and Create new db $DBNAME ..."

dropdb -U $DBUSER -h $DBHOST -p $DBORT $DBNAME
createdb -U $DBUSER -h $DBHOST -p $DBORT $DBNAME
psql -U $DBUSER -h $DBHOST -p $DBORT $DBNAME <<EOF
create extension postgis;
create extension pgrouting;
create schema data;
alter database $DBNAME set search_path to data, public;

EOF

echo 'Loading data ...'
./load-data-for-pgrouting.sh $DBNAME

echo 'Preparing data for routing ...'
psql -U $DBUSER -h $DBHOST -p $DBORT $DBNAME -f prep-for-pgrouting.sql
psql -U $DBUSER -h $DBHOST -p $DBORT $DBNAME -f reassign-vertix-ids-3d.sql

echo 'Adding Driving Directions code ...'
psql -U $DBUSER -h $DBHOST -p $DBORT $DBNAME -f explicate.sql
psql -U $DBUSER -h $DBHOST -p $DBORT $DBNAME -f explicate_arb.sql
psql -U $DBUSER -h $DBHOST -p $DBORT $DBNAME -f explicate_eng.sql
psql -U $DBUSER -h $DBHOST -p $DBORT $DBNAME -f explicate_esp.sql
psql -U $DBUSER -h $DBHOST -p $DBORT $DBNAME -f explicate_fre.sql
psql -U $DBUSER -h $DBHOST -p $DBORT $DBNAME -f imaptools_routing_wrappers.sql
psql -U $DBUSER -h $DBHOST -p $DBORT $DBNAME -f analyze-network.sql
#psql -U $DBUSER -h $DBHOST -p $DBORT $DBNAME -f dd-query.sql
```

## How does it work?

The results of a pgRouting query returns a set of records representing each road segment in order through the network from the starting position to the ending position. To generate driving directions we have to turn this list of edges into maneuvers, then we have to turn the maneuvers into natural language expressions the explicate each manuever.

### What is a Manuever?

A manuever is an event where we need to do something while driving, like make a turn, arrive at the destination, enter or exit a tunnel, depart the starting location, head a compass direction, etc. If you are driving down Main St, it is likely multiple segments all with the same name and we don't want to explicate "Continue on Main St for 0.2 miles" at every intersection along Main St. so we have to combine multiple edges with the same street name into a single maneuver like, "Continue on Main St for 2.5 miles". When it comes to turns at an intersection, it is helpful to indicate right or left but also if the turn is slight or sharp or a u-turn in that direction. If round-abouts are well defined in the data, we might want to indicate that we want the 1st, 2nd, or nth exit from the round-about from where we entered it.

The current code might not handle all these and other conditions, but the basic structure is extendable to support any conditions that you can analyze and define as a manueuver.

### Explication

Given that we have a set of manueuvers, explication is done by mapping the manuever to a natural language template and then substituting strings into the template. We define a schema for each language we want to support, the the explication code does the string substitution and we get natural language phrases in any language you have translated the templates into. Currently we have templates for Arabic, English, French and Spanish.


