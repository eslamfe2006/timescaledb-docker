This image based on stolon image sorintlab/stolon:master-pg10 which allow you to work with high available postgres cluster on kubernetes which include timescale 1.4 on it

just you need only to add the extension on the required database 

first # Connect to PostgreSQL, using a superuser named 'postgres'

psql -h "pod-ip" -U stolon -d postgres

Now create a new empty database (skip this if you already have a database):

-- Create the database, let's call it 'tutorial'
CREATE database tutorial;


-- Connect to the database
\c tutorial

-- Extend the database with TimescaleDB
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

you can run it from stolon repository on helm chart  
https://github.com/helm/charts/tree/master/stable/stolon
just change the image name and tag to eslam91mohsen/timescale-stolon-ha:pg10-1.4

note: if you want to use old version (1.3) from timescale you can change image tag to pg10-7