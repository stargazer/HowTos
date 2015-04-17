## Convert all database tables of a Django application, from MyISAM to the InnoDB storage engine

### Assumptions

* The application can tolerate some downtime

### Description   
Simply running ``ALTER`` statements to convert the database tables from ``MyISAM`` to ``InnoDB`` won't work. While the tables will indeed be converted to ``InnoDB``, their foreign key's will remain as simple references, instead of real foreign keys in the way ``InnoDB`` implements them. The reason is that Django, when first created those tables, it defined the foreign keys using ``MyISAM``'s ``KEY`` statement rather than ``InnoDB``'s ``CONSTRAINT`` statement. So even though we could convert the tables to ``InnoDB``, we'll still don't get real ``InnoDB`` foreign keys.

So, on a higher level what we need to do is the following:

* Ensure that all future tables will be using the ``InnoDB`` storage engine
* Stop the webserver or any other processes that might be initiating database queries    
* Take a database dump without including the statements to recreate the tables (since these contain ``MyISAM`` definitions)
* Drop all tables
* Recreate all tables (using ``InnoDB``)
* Load the database dump
* Restart the webserver or any other process we disabled earlier    

### Steps



#### Ensure that all future database tables will be using the ``InnoDB`` storage engine
  
  Edit your Django application's ``settings.py`` file, and make sure that the default storage engine is set to ``InnoDB``

        DATABASES = {
            'default': {
                ...
                'OPTIONS': {
                    'init_command': 'SET storage_engine=InnoDB',
                }, 
            }        
        }        

  What this setting ensures is that any **new** database tables created from now on, will be using the ``InnoDB`` storage engine. It doesn't affect the already existing database tables. 

#### Stop the webserver and any other processes that might be initiating database queries  

    sudo service apache2 stop

or

    sudo service nginx stop

#### Take a database dump

Use ``mysqldump`` and ensure that:

* You do not include queries for recreating the DB tables (``no-create-info``), as this would essentially record queries for recreating the ``MyISAM`` tables
* Use complete ``INSERT`` statements (``--complete-insert``), so that each INSERT query will specify exactly which column takes each value
* Use ``REPLACE INTO`` instead of ``INSERT INTO`` (``--replace``), so that duplicates won't blow up the data restore later ([When and how can duplicates appear?](#when-and-how-can-duplicates-appear))
   
        mysqldump -uUSERNAME -pPASSWORD --complete-insert --replace --no-create-info DATABASE > BACKUP

where ``USERNAME`` is the database user, ``PASSWORD`` is the corresponding password, ``DATABASE`` is the database name and ``BACKUP`` is the backup file.

#### Drop all database tables

    mysqldump -uUSERNAME -pPASSWORD --add-drop-table --no-data DATABASE | grep -e '^DROP \| FOREIGN_KEY_CHECKS' | mysql -uUSERNAME -pPASSWORD DATABASE

#### Re-create all database tables and run all migrations
    
    manage syncdb --noinput
    manage migrate --all

where ``manage`` is the Django application's management script.

Since we earlier specified ``InnoDB`` to be the default storage engine for all newly created database tables all these newly created tables will be using ``InnoDB``, and have real foreign key constraints where needed.

Note that at this point some of the tables we just created, might contain records. See [When and how can duplicates appear?](#when-and-how-can-duplicates-appear)

#### Restore the database backup
  
    mysql -uUSERNAME -pPASSWORD DATABASE < BACKUP

#### Restart the webserver or any other process that we disabled earlier

    sudo service apache2 restart

or

    sudo service nginx restart

### Sample

You can see a gist [here](https://gist.github.com/stargazer/2ee2f9f6e158616b0ef5)

### Tips
    
Upgrade to MySQL >= 5.5 so that ``InnoDB`` will be the default storage engine anyway. This will also help avoid the ``init_command`` in Django's ``settings.py``'s ``DATABASES`` parameter, which causes some [overhead](https://docs.djangoproject.com/en/1.8/ref/databases/#creating-your-tables) on every database connection.
 
### When and how can duplicates appear?

When running Django's management scripts ``syncdb`` and ``migrate`` that create the database tables and apply all migrations, it might be the case that some tables will end up containing some records. A typical example is the table ``auth_permission`` of ``django.contrib.auth``. Other examples might be data migrations that prepopulate tables with a few records.

In the case that there are indeed records in these tables, when later restoring the database, the ``INSERT``s for those specific pre-existing records will fail, and the import will be interrupted. 

The backup we created doesn't contain queries to ``DROP`` and ``CREATE`` the tables, since we used the ``--no-create-info``. Rather we manually create them later. So at the time of restoring the database backup, the database tables have already been created and probably contain some records. However, since the backup uses ``REPLACE`` instead of ``INSERT`` statements (because of using ``--replace`` flag on ``mysqldump``), when restoring the database backup the existing records will silently be replaced, and the data import won't get interrupted.
