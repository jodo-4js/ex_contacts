# Genero database synchronization demo "Contacts"

## Description

### Introduction

This Genero BDL demo shows how to implement mobile / server database
synchronization with a REST Web Service.

The mobile app has its own local database, and can share contact data
with other mobile apps throught the central database program.

Allow user/contact location with Google geolocation services to show
a map with contacts nearby, based on the contact address.

Note that the contact database of this demo is isolated: it does not
use the contact database of the mobile device.


### Components

1. Mobile:
  * the contacts app (for Android or iOS)
2. Backend:
  * the database creation program
  * the configuration program to define app users and data filters
  * the dbsync server program implementing the REST Web Service


### Key features

1. Optimized DB Synchronization process:
  * based on timestamp modification flags
  * only modifed contact records are sent
  * does not send contact photo data if not needed
  * does not send contact notes details if not needed
  * data filter can be defined per users (to load only contacts from a given city or country)
  * return receipt mechanism to make sure mobile app has updated local database
2. DB Synchronization can be encrypted over HTTPS, using JSON or XML format.
3. Server programs can be executed through GAS for load balancing.
4. App users can protect their login with a password.
5. Conflicts are managed on the server side, sync report displayed if conflicts.
6. Users can be activated/deactivated from server config program.
8. In case of de-sync, can do a full-sync to cleanup and retrieve all contacts from server.
9. Automatic refresh with timeout config.
10. Send GPS coordinates for current user.


## Requirements

* Database server: Tested with Informix 11.20 and PostgreSQL 9.3.
* FGLGWS development environment >= 3.00 (for the dbsync server process)
* Genero Mobile (GMI/GMA) >= 1.20
* GNU make utility or Genero Studio >= 3.00


## The server database (UTF-8)

For production, it is better to use a real multi-user DB such as Informix
or PostgreSQL.

However, for testing purpose, you can use the default SQLite database,
which is read-to-use, in server/contacts.sqlite.

To create a UTF-8 database on the server with Informix:

```
$ echo $DB_LOCALE
en_us.utf8
$ dbaccess - -
> create database contacts with buffered log;
```

To create a UTF-8 database on the server with PostgreSQL:

```
$ createdb -h localhost contacts --encoding "utf-8"
```

## Using Genero Studio projects

### Server-side programs

* Open a first Studio instance and load the *server_progs.4pw* project file.
* Make sure you have a valid Genero Desktop configuration.
* Configure UTF-8 locale with FGL_LENGTH_SEMANTICS=CHAR (in build rules env vars)
  * **Warning**: On Windows you need to set LANG=.fglutf8 (remove LC_ALL)
* Build the server programs.
* Create your server database if needed (you can also use the ready-to-use SQLite database)
  1. Go to the *mkcontacts_main* application node.
  2. Edit the command line arguments in order to use your database.
  3. Rune the *mkcontacts_main* program to create the database tables.
* Configure the server database:
  1. Go to the *server_config* application node.
  2. Edit the command line arguments in order to use your database.
  4. Run the program.
* Execute the server program managing database synchronization:
  1. Go to the *dbsync_contact_server* application node.
  2. Edit the command line arguments in order to use your database.
  3. Check the command line arguments for the TCP port.
  4. Run the program.

### Contacts app for mobile

* Open a second Studio instance and load the contacts.4pw project file.
* Configure UTF-8 locale with char length semantics (build rules env vars) (on Windows you need to change LANG to .fglutf8)
* Configure the environment for Android or iOS.
* Build the app and deploy the contacts app.


## Using command line (Linux)

### Setup Genero BDL environment

Setup Genero BDL 3.00+ environment

```
$ fglrun -V
fglrun 3.00 .....
```

Define the correct locale and length semantics: UTF-8 + CLS

```
$ export LC_ALL=en_US.utf8
$ export FGL_LENGTH_SEMANTICS=CHAR
```


### Compile all programs

Set the TOP environment variable to the root directory of the sources:

```
$ cd <topdir>
$ export TOP=$PWD
```

Execute the top makefile

```
$ make clean all
```


### Create database tables on the server

If you want to create your own database instead of using the default
SQLite database provided in server/contacts.sqlite, you must create
the database tables and fill with sample data.

The sample data is created for 3 users: mike, max, ted ...

See mkcontacts.4gl source code for details.

To create the database tables:

```
$ cd <topdir>
$ make
$ export FGLLDPATH=$PWD/common
$ cd server
$ fglrun mkcontacts_main -d contacts -o <driver> -u <user> -w <pswd> -s
```

WARNING: Use -s option to create with sample data!!!!!!!


### Start the dbsync server

```
$ cd <topdir>
$ make
$ export FGLLDPATH=$PWD/common
$ cd server
```

Then, to start the server with your own database:

```
$ fglrun dbsync_contact_server -v -d contacts -o <driver> -u <user> -w <pswd> -p <port>
```

Or, to start the server program with the default SQLite database:

```
$ fglrun dbsync_contact_server -v -p <port>
```

Note the -v option to use the verbose mode.

Now the server program runs in standalone mode. See below how to configure
the GAS to run dbsync server programs in an application server context,
to get load balancing.

To check if the server is running, open a web browser and enter followin URL:

```
http://localhost:<port>/ws/r/dbsync_contact_server/mobile/dbsync/status
```

Note: The server program can automatically query the google geolocalization service
to set GPS coordinates from the contacts addresses. In order to enable this
feature, you need to register to this google service and get a API Key.
Then start the dbsync_contact_server program with the -k <google-api-key> option.


### Deploying the dbsync server in a GAS

1. Install GAS and set the environment for GAS.
2. Create the .xcf file for the server program (see dbsync_contact_server.xcf)
   Following environment variables must be defined in the .xcf file:
    * LC_ALL=en_US.UTF-8
    * FGL_LENGTH_SEMANTICS=CHAR
    * LD_LIBRARY_PATH to DB client used
    * FGLLDPATH to find modules in ./common
    * Put the correct path to the dbsync server program (<PATH>)
    * Define the parameters for the dbsync server program (<PARAMETERS>)
3. Copy the .xcf file to $FGLASDIR/appdata/services
4. Start the GAS (httpdispatch)
5. Check the GAS config with a browser:
 ```
 http://localhost:6394/ws/r/dbsync_contact_server/mobile/dbsync/status
 ```
 If the browser does no show a welcome page, check the GAS logs.
6. Check with the application on the mobile device, by entering the URL:
 ```
 http://<server_host>:6394/ws/r/dbsync_contact_server
 ```


### Compile the mobile application

Setup GMA or GMI app build tool.

```
$ cd <topdir>
$ make all
$ cd app
$ make appdir
```

For Android:

```
$ make package_gma
```
or

```
$ sh build_gma.sh 
```

See shell for required settings and generated APK.

For iOS:

```
$ make package_gmi
```

or

```
$ sh build_gmi.sh
```

See shell for required settings and generated IPA.


### Starting the mobile app

The first time you start the app, the SQLite database will be created.

When running the app on a device, no specific configuration is required.

When running the app from a server in dev mode, you can specify the
SQLite database directory for the user with the USERDBDIR env var, and
this directory must exist:

```
$ cd <topdir>
$ make
```

Create a directory for the app database for use ted for example:

```
$ mkdir /tmp/dbdir_ted
$ export USERDBDIR=/tmp/dbdir_ted
```

Go to the appdir directory:

```
$ cd <topdir>/build/appdir
```

Set FGLPROFILE:

```
$ export FGLPROFILE=$PWD/fglprofile
```

Set FGLIMAGEPATH to find application icons (image2font.txt), default
image files ($PWD/images) and application pictures ($PWD, to find
images in the $PWD/bfn_tmp directory):

```
$ export FGLIMAGEPATH=$FGLDIR/lib/image2font.txt:$PWD/images:$PWD
```

Set the IP address of the mobile device:

```
$ export FGLSERVER=<mobile-device-IP>
```

Run the program:

```
$ fglrun main
```

To cleanup, consider removing the SQLite database created in dbdir_ted.


### Deploying the app on an Android emulator

When running the contacts app in an emulator from the Android SDK, the IP
address of the host machine when the dbsync server runs is:

```
10.0.2.2     Special alias to your host loopback interface
                (i.e., 127.0.0.1 on your development machine) 
```

This is the address you should enter in the "Host" field in the contacts
app settings.

http://developer.android.com/tools/devices/emulator.html


## Usage

After compiling server programs and deploying the mobile app:

* Start the server program dbsync_contact_server is started.
* Configure the users with the server_config program.
  * Add users if needed.
  * Define data filters.
* Make sure your mobile devices is one the same Wifi as the server.
* Start the app on the mobile device.
* At first start, the app will ask for config settings.
  * Define the server Host IP address
  * Define the port if you have changed it on the server side.
  * Define the user id (ted, mike or max are predefined)
  * Configure the GAS settings if the server program is behing GAS.
  * Tap the "Test" button to see if the connection can be established.
  * Tap OK to save and close.
* First synchronization should occur.
* Modify, add, delete contacts.
* To sync, tap "Options" + "Synchronize".
* Start the app in a second device with a different user.
* If data becomes de-synchronized for some reason, perform a full sync with "Options" + "More" + "Full sync".
* Using GPS / localization feature:
  * The server program must have been started with a Google API Key to use geolocation services.
  * Associate the app user defined by the user id to a contact (yourself):
    Tap on a contact to modify it, "Options" + "Bind user", validate.
  * In main list, try "Options" + "Localize" to get the map.
  * The device must have GPS activated.
  * When a contact is modified and synchronized, the server program returns the position from the contact address.


## Todo list

* When feature is available, produce smaller photo files with choosePhoto/takePhoto front calls
