# Partial Database Restore

## Scenario

* We keep daily backups of our application's production database
* The database's schema is as such:

              -------------
              |   client  |
              -------------
               ^        ^
              /          \
             /            \
          -----------   --------	     
          | contact |   | list |
          -----------   --------
              ^           ^
               \         /
                \       /
            --------------	    
            | membership |
            --------------	    

* A few hours after our latest backup, one of our clients has accidentally deleted his address book (all his ``contact`` records, and together, all his ``membership`` records)
* Tables ``client`` and ``list`` remained intact.

## Question

How can we restore the address book of this specific client, without affecting any of the other clients' data, as well as the rest of this client's data?

## Solution

Apparently we can't simply restore the database using our latest backup, as this will affect all the latest changes of all our users. What we need to do is use this database backup to restore all the ``contact`` records that have a Foreign key to the affected ``client``, and all ``membership`` records that have a Foreign Key to a client ``contact``.

So basically, 2 things:

* Use the latest backup to export only this client's ``contact`` and ``membership`` records to a file
* Use this file to import the records to the production database

### In detail

1. First load the latest production database backup on some local database
2. Construct a ``SELECT`` query that returns this client's ``contact`` records:
       
         SELECT contact.id, contact.client_id, contact.name, contact.surname 
         FROM contact 
                 INNER JOIN client 
                 ON contact.client_id = client.id
         WHERE client.id=``client_id``

  This query will be the basis of the export of the client's contacts. 

3. The export it self can be achieved using MySQL's ``SELECT ... INTO OUTFILE`` syntax. Let's squeeze this into the query we constructed above:

         SELECT contact.id, contact.client_id, contact.name, contact.surname
         INTO OUTFILE '/tmp/contacts.db'
         FIELDS TERMINATED BY '\t'
         LINES TERMINATED BY '\r\n'
         FROM contact 
                INNER JOIN client 
                ON contact.client_id = client.id
         WHERE client.id=``client_id``

   Running the query will create file ``/tmp/contacts.db``. This file includes all records selected. Fields are separated by a ``\t`` character, and records separated by ``\r\n``.

   > Note: Make sure that your MySQL user has been granted the [FILE](http://dev.mysql.com/doc/refman/5.1/en/privileges-provided.html#priv_file) privilege, and that the MySQL process has write permissions on the ``/tmp`` folder.

4. Now we need to copy this file to the production server, and get ready to load it into the production database. So, on the production database, construct this query:

        LOAD DATA INFILE '/tmp/contacts.db' IGNORE 
        INTO TABLE contact  
        FIELDS TERMINATED BY '\t'  
        LINES TERMINATED BY '\r\n' 
        (id, client_id, name, surname)

   This query reads the file ``/tmp/contacts.db``. We've declared that each record is terminated by the ``\r\n`` character, fields are terminated by the ``\t`` character, and that the fields of every record are ordered as ``id``, ``client_id``, ``name``, ``surname``. The ``IGNORE`` keyword means that if a record with the given ``id`` already exists, it won't be replaced. Alternatively we can use the ``REPLACE`` keyword.

   Running this query will load all the ``contact`` records into the production database.

   > Note: Again make sure that your MySQL user has been granted the ``FILE`` privilege, and that the process can read from the ``/tmp`` folder, or whatever folder you copied the file to.

5. Go back to Step 2, and repeat for the ``Membership`` table

6. Thank God



