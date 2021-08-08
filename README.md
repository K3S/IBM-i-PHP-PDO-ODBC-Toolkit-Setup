# Purpose

I have created this guide as a reference for myself, but also to help others who might be trying to accomplish the same thing. Much of this info is copied and pasted, and I do not represent it as my own, only that I have aggregated it into one place for ease-of-use (there were many different sources for all of this). 

Notably, thank you to [Chuk Shirley](https://github.com/chukShirley){:target="_blank" rel="noopener"}, [Stephanie Rabbini](https://twitter.com/jordiwes){:target="_blank" rel="noopener"}, [Alan Seiden](https://twitter.com/alanseiden){:target="_blank" rel="noopener"}, [Dave Dressler](https://godzillai5.wordpress.com/){:target="_blank" rel="noopener"}, and [Kevin Adler](https://twitter.com/kadler_ibm){:target="_blank" rel="noopener"}. 

# Synopsis (What The Heck Does This All Mean??)

[IBM i OS](https://en.wikipedia.org/wiki/IBM_i) on [IBM Power Systems](https://en.wikipedia.org/wiki/IBM_Power_Systems) hardware is an ABSOLUTELY amazing collection of technologies with true rock solid stability and features to help with enterprise level challenges. However, the perception of the platform (Usually referred to as [AS/400 or iSeries](https://en.wikipedia.org/wiki/IBM_System_i)) is that the technology is dated and not suited for the modern business environment.

This is 100% innaccurate and not true, and mostly comes from the command line interface users would be expected to use ([IBM 5250](https://en.wikipedia.org/wiki/IBM_5250)) and/or some of the programming quirks that arise from being able to compile and run all code since its inception. 

Instead there are options to leverage the IBM i OS to its absolute strength while employing modern techniques / interfaces so that users. 

The power of [IBM RPG](https://en.wikipedia.org/wiki/IBM_RPG) is that it is write once, compile, and run on the platform 'forever'. No matter the OS upgrades and new hardware any of your code written in RPG / CL will run on the new system. If used correctly this can lower your future technical debt and allow a lot more 

For your consideration I would suggest following (or putting into place with your current applications) the [CQRS](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation) design philosphy to your application. We ([K3S](https://k3s.com)) have modernized our application by taking every action done within a database that is not a query (the COMMAND action within CQRS, which usually represents a Create, Update, or Delete of a CRUD application) and created single purpose APIs to accomplish a task called with the [IBM i PHP Toolkit](https://www.seidengroup.com/toolkit/). For example: if within our application we wanted to create a new supplier, we would call the [ADDSUPL API](https://technical.k3s.com/docs/api/SUPL/#addsupl---add-supplier). This API will take in the values needed to create a supplier, and if all the values are valid (meaning there can be a lot of business logic that is encapsulated within the API) it will be successful, or fail and return a valid message. This means these business logic APIs, once compiled and working, will be viable for a long time. 

For the Query portion of CQRS you can then use whatever preferred interface language you want. Below we have used PHP to accomplish this action. It has allowed us to use a modern PHP framework called [Laminas](https://getlaminas.org/) (A community-supported, open source continuation of Zend Framework), modern styling via [Bootstrap](https://getbootstrap.com/), and the flexibility that if we want to change our interface to a different language (though we love PHP!). 

This guide was written with these ideas and goals in mind and we hope others use them to gain the full potential of IBM i!

Below I have written out instructions for 4 distinct scenarios:
* Running PHP on your IBM i with Apache as the web server
* Running PHP on your IBM i with NGINX as the web server
* Running PHP on another server (Windows or Linux) but connecting back to your IBM i DB2 database
* Calling RPG / CL programs on your IBM i from PHP running on another server (Windows or Linux)

# Installing PHP RPM on IBM i

This is a guide on how to install the PHP RPMs from Zend. This is the community edition you can use as any normal open source software. The ODBC extension is not installed with the normal Zend Server version of PHP, so if you want to use ODBC with PHP you are going to need to install the RPMs. Zend Server and PHP RPMs are pragmatically no different from a run time perspective. Most of the advantage in Zend Server is in the development, going through logging, and they have a custom package management. As well the intl and zip extensions are not available via the RPMs. Otherwise most shops should just consider the PHP RPMs. 

1.	Setup Package Manager: Make sure the you have installed the Open Source Package Management (OSPM) from ACS [Getting started with Open Source Package Management in IBM i ACS](https://www.ibm.com/support/pages/getting-started-open-source-package-management-ibm-i-acs){:target="_blank" rel="noopener"}

If you do not have SSH working to setup OSPM, these instructions will allow you to install [OSPM from ACS](https://ospm.k3s.com){:target="_blank" rel="noopener"}

2.	Install yum utilities: From the OSPM install yum-utils from the Available Packages tab. This will allow you to add 3rd party packages. 

3. Install PHP RPMs: From a shell command line (I recommend SSHing in to the server, but I believe you can use QSH) install the repo to PHP RPMs hosted by Zend. Here is a list of 3rd Party RPMs (make sure you read the note below before you add PHP). All of the open source software shoudl be in the '/QOpenSys' directory and yum specifically in '/QOpenSys/pkgs/bin'.

   [3rd Party Open Source Repos for IBM i](https://ibmi-oss-docs.readthedocs.io/en/latest/yum/3RD_PARTY_REPOS.html){:target="_blank" rel="noopener"}

   ```
   yum-config-manager --add-repo http://repos.zend.com/ibmiphp/
   ```
   
   The command I ran because of where yum is located
   
   ```
   /QOpenSys/pkgs/bin/yum-config-manager --add-repo http://repos.zend.com/ibmiphp/
   ```

   As the repo RPMs for PHP were now added to ACS, now, just as you added yum-utils from the Available Packages tab, add the PHP packages / extensions you want. I would just add all of them that begin with php. They are not very large and you will end up probably using all of them.

4. Configuring PHP: Mostly you will use the defaults already setup, but you can configure PHP now to fit your environment. You will need to move the php.ini file into the php sub directory to use the .ini file.  
   
   The default php.ini file is located in this directory:
   ```
   /QOpenSys/etc/php.ini
   ```

   This must be moved to the php subdirectory to be loaded by PHP RPMs:
   ```
   /QOpenSys/etc/php/php.ini
   ```

   Extensions are enabled via this directory: 
   ```
   /QOpenSys/etc/php/conf.d
   ```
   
   Check that your php.ini file is loaded by creating an index.php file in your default htdocs directory with the below configuration output:
   ```
   <?php phpinfo(); ?>
   ```
         
   Visit [PHP.net](https://php.net){:target="_blank" rel="noopener"} to learn about configuration options. 
   
   Default versions of configuration files and all the extensions are included when you install the RPMs.
   
   Updates to RPMs: If you have made changes to the configuration files, subsequent RPM updates will preserve your changes. If you are running the config files unchanged from install it will install the newer versions. But if you made changes to the config files it will keep the versions you have and write the new default ones to the same directory with this naming scheme: _filename_.rpmnew

# Setup Apache to Run PHP

1. Create New Apache Instance On IBM i: Visit http://ibmiipaddress:2001/HTTPAdmin on your server where you replace ibmiipaddress with the IP address of your IBM server (note that the [Admin Server](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_72/rzaie/rzaiemngsrvr.htm){:target="_blank" rel="noopener"} must be running). Click 'Create HTTP Server' in the top left hand corner, and go through the steps of adding a new Apache server. Note the webroot folders if you adjust from defaults. 

2. Set Up Aache To Run PHP:

   Once you have created the Apache instance, use the "Edit Configuration File" option on the
left panel to add the following configuration options near the top:

   ```
   # This loads the Apache FastCGI support originally created for Zend
   LoadModule zend_enabler_module /QSYS.LIB/QHTTPSVR.LIB/QZFAST.SRVPGM

   # This tells Apache that any file with a .php extension should be
   # executed by the FastCGI application/x-httpd-php handler
   AddType application/x-httpd-php .php
   AddHandler fastcgi-script .php

   # Let's you go to http://example.com instead of http://example.com/index.php
   DirectoryIndex index.php index.html
   ```

   Once you have added these configuration options, click the OK button at the bottom
of the page to save it.

3. Configure FastCGI: Now that Apache is set up for FastCGI, we need to configure a FastCGI handler
for application/x-httpd-php. Without this, the FastCGI processor won't work
and the PHP script will merely be downloaded by the web browser.

   Create a file called fastcgi.conf in `/www/<server name>/conf` (assuming the
default path for the webroot was chosen) with the following contents:

   ```
   Server type="application/x-httpd-php" CommandLine="/QOpenSys/pkgs/bin/php-cgi" StartProcesses="1"
   ```

   You can now start the web server.

4. Test: Create a small index.php file in your webroot with the following code:

   ```
   <?php phpinfo(); ?>
   ```

   And visit the virtual host you set up. You should see the PHP info page. If you do, you are running PHP via RPM on your IBM i. 

5. Recommended Additional Configuration Options For Apache, Speed, and a Common Issue: While your mileage may vary, I recommend these additional lines for your consideration in your Apache config to speed up your app. 

   ```
   # These lines will add gzip compression to the data served. You want these near the top
   LoadModule deflate_module /QSYS.LIB/QHTTPSVR.LIB/QZSRCORE.SRVPGM
   AddOutputFilterByType DEFLATE application/x-httpd-php application/json text/css application/x-javascript application/javascript text/html

   # This will turn on Keep Alive and allow users to reuse older connections, making the serving of data faster. 
   TimeOut 30000
   KeepAliveTimeout 30
   HotBackup Off

   # If your code looks funky a lot of times it is your CCSID. This seems to help
   DefaultFsCCSID 37
   CGIJobCCSID 37
   ```

# Setup NGINX To Run PHP

## Notes on NGINX and PHP-FPM

NGINX has built-in support for proxying requests to a FastCGI process, but it does not provide a built-in FastCGI process manager. For that, we will be using the standard PHP-FPM utility.

NGINX has a good document on setting up [WordPress in NGINX](https://www.nginx.com/resources/wiki/start/topics/recipes/wordpress/). Since WordPress is written in PHP, this can be used as the basis for setting up PHP with NGINX on IBM i.

### Setting up NGINX

Install NGINX from the Open Source Package Manager. 

We will then need to creat an NGINX configuration. As there is no wizard this will need to be done by hand. Create a configuration under `/QOpenSys/etc/nginx/` called php.conf

In this example, we will mimic the file structure of an Apache webserver on IBM i:

- /www/php: base root
- /www/php/logs: logs directory
- /www/php/htdocs: web server root


```
error_log  /www/php/logs/nginx.err.log;
pid        /www/php/logs/nginx.pid;

events {
    # required section, just leave empty for defaults
}

http {
    upstream php {
        # this is the default FPM listen address
        server 127.0.0.1:9000;
    }

    server {
        # set your actual hostname here
        server_name mywebserver.example.com;

        # set your listen port and address here
        listen 6090 default_server;

        # document root for web server
        root /www/php/htdocs;

        # Let's you go to http://example.com instead of http://example.com/index.php
        index index.php index.html;

        # If you want this to be case insensitive, you will add an * after the ~
        # Example: location ~* \.php$ {
        location ~ \.php$ {
            include /QOpenSys/etc/nginx/snippets/fastcgi-php.conf;

            # proxy to the php upstream we defined earlier
            fastcgi_pass php;
        }
    }
}

```

We also need to create the PHP FastCGI snippet referenced above:

```
fastcgi_split_path_info ^(.+\.php)(/.+)$;

# Check that the PHP script exists before passing it
try_files $fastcgi_script_name =404;

# Bypass the fact that try_files resets $fastcgi_path_info
# see: http://trac.nginx.org/nginx/ticket/321
set $path_info $fastcgi_path_info;
fastcgi_param PATH_INFO $path_info;

fastcgi_index index.php;
include fastcgi.conf;
```

### Setting up FPM

Now that NGINX is configured, we need to set up FPM. Luckily, FPM is mostly configured ok out of the box.

The main FPM config file is located at `/QOpenSys/etc/php/php-fpm.conf`, which really just serves as a way to load `/QOpenSys/etc/php/php-fpm.d/www.conf`.

Again, the defaults should suffice, but one thing that is tricky is that FPM wants to set a user *and* group to run under. The default user has been set to QTMHHTTP, but this user is not a member of any groups, which FPM detects as an error:

```text
[19-Jul-2019 13:27:35] ERROR: [pool www] please specify user and group other than root
[19-Jul-2019 13:27:35] ERROR: FPM initialization failed
```

There are two options:

1. Modify QTMHHTTP user to have a primary group, eg. `CHGUSRPRF USRPRF(QTMHHTTP) GRPPRF(GRP1)`
2. Configure FPM in `/QOpenSys/etc/php/php-fpm.d/www.conf` to run under a given user profile, eg. `group = grp1`


### Starting Things Up

Once everything is configured, you can start everything:

- Start NGINX: `nginx -c php.conf`
- Start FPM: `/QOpenSys/pkgs/sbin/php-fpm`

Your NGINX server will be running under your user profile and FPM will be running under the user profile specified in the config.
 
## Testing the Setup

Finally, we need a PHP script to run. Create an index.php in the document root
for the web server, eg. `/www/php/htdocs`:

```
<?php echo phpinfo(); ?>
```

# Example ODBC Connection To DB2 On IBM i

These are some example connection strings and directions to help people connect via PDO / ODBC to DB2 on IBM i. These examples include running a PHP application on a Linux / Windows server OR running PHP directly on IBM i. 

## What is PDO and ODBC?

PDO (PHP Database Object) is an abstracted database connection developed for PHP. By using PDO you can write one generalized query that can 'run anywhere' once connected. 

ODBC is a standard database connection method developed to allow applications to connect the 'same way' to any database. 

By using PDO and ODBC to connect to DB2 on IBM i you can use generalized methods to develop your application. 

## IBM Connection String Reference

This is the collection of options to help you buid your ODBC string. 

[IBM ODBC Connection String Keywords](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_74/rzaik/connectkeywords.htm){:target="_blank" rel="noopener"}

## Important PHP Setting

It is important to set in the php.ini OR the .user.ini file this ODBC setting:

odbc.default_cursortype=0

This sets the cursor type to Forward Only Cursor, and can provide a huge speed improvement for large amounts of data. 

[ODBC PHP Configuration](https://www.php.net/manual/en/odbc.configuration.php#ini.uodbc.defaultcursortype)

## Connecting A PHP Application Running On IBM i To A DB2 Database Running On IBM i 

This is an example of how to use PDO and ODBC to connect to DB2 on IBM i when PHP is running on IBM i. This will NOT work by default with Zend Server PHP as they do not include the necessary ODBC extension. You must either add the extension or run and install the PHP RPMs listed above. From a runtime perspective using ODBC there is no difference between the two. Zend Server has some nice debugging tools and a set way to deploy applications, and some extensions are not available (intl and zip) via the RPMs. Otherwise most shops should consider just running the PHP RPMs.

I have found the order of the install of the next two pieces matter, so install step 1, then step 2. 

1. You will need to install the unixODBC and unixODBC-devel from the OSPM. These drivers, along with the PASE IBM i ODBC driver will allow your app to connect to the DB2 database.

2. Next, while the IBM i OS has a built in ODBC server to accept connections by default, it does not have an ODBC client driver installed by default. You will need to download the [PASE IBM i ODBC](https://www.ibm.com/support/pages/odbc-driver-ibm-i-pase-environment){:target="_blank" rel="noopener"} driver and install. 

The directions will mention setting up a DSN within odbc.ini or the user odbc.ini. You can either set up your database connections this way or configure your odbc connection via a string as shown below. This approach allows you to track your connection configuration in your git repository.

- NAM=1; This is the \*SYS naming convention  
- TSFT=1; This sets the timestamp type to IBM standards

```
<?php
/*
 * Database connection information: https://docs.zendframework.com/zend-db/adapter/
 */
return array (
    'db' => [
        'dsn' => 'odbc:DRIVER={IBM i Access ODBC Driver};SYSTEM=ipaddress;UID=ibmiusername;PWD=ibmipassword;NAM=1;TSFT=1;DBQ=, THIS IS WHERE YOU PUT THE LIBRARY LIST THE COMMA IN FRONT SAYS NO DEFAULT LIBRARY',
        'driver' => 'Pdo',
        'platform' => 'IbmDb2',
        'platform_options' => [
            'quote_identifiers' => true,
        ],
        'driver_options' => [
            PDO::ATTR_PERSISTENT => true,
            PDO::ATTR_EMULATE_PREPARES => true,
        ],
    ],
);
```


## Connecting A PHP Application Running On Linux / Windows To A DB2 Database Running On IBM i 

As PHP can run on multiple OSes, it can be beneficial in some circumstances to run PHP on another server and use IBM i just for its DB2 database and business logic (for example, calling RPG via the PHP Toolkit). 

```
<?php
/*
 * Database connection information: https://docs.zendframework.com/zend-db/adapter/
 */
return array (
    'db' => [
        'dsn' => 'odbc:DRIVER={IBM i Access ODBC Driver};SYSTEM=ipaddress;UID=ibmiusername;PWD=ibmipassword;NAM=1;CCSID=1208;DBQ=, THIS IS WHERE YOU PUT THE LIBRARY LIST THE COMMA IN FRONT SAYS NO DEFAULT LIBRARY',
        'driver' => 'Pdo',
        'platform' => 'IbmDb2',
        'platform_options' => [
            'quote_identifiers' => true,
        ],
        'driver_options' => [
            PDO::ATTR_PERSISTENT => true,
            PDO::ATTR_EMULATE_PREPARES => true,
        ],
    ],
);
```

## Releasing Locks On Files Created From ODBC Connections

When accessing files via the ODBC connection there are jobs created called QZDASOINIT. These jobs can lock files and, at some point, you might need to release the locks (especially if you use the Lazy Close option to speed up response time) They appear to naturally go away after 15-30 minutes, however if you need instant release we have open sourced some code to help. 

[Release Locks From ODBC Connections](https://github.com/K3S/IBMi-Utilities/blob/master/ReleaseLocks/endjob.sqlrpgle){:target="_blank" rel="noopener"}

## Calling RPG via The PHP Toolkit Over PDO ODBC When PHP Runs On Linux / Windows

It is possible to call RPG from another server over ODBC. This uses your PDO connection referenced above. Below is the code from my application running in Zend Framework. Notice on instantiation of the toolkit (the new Toolkit line) I am passing the current database connection, and the fourth parameter is 'pdo'. This is allowing the Toolkit to use our current connection resource from the PDO object over ODBC to call RPG on the IBM i (yes this is ridiculously cool). 

[IBM i PHP Toolkit Repo](https://github.com/zendtech/IbmiToolkit){:target="_blank" rel="noopener"}

```
<?php

namespace RPG\Service\Factory;

use Zend\ServiceManager\Factory\FactoryInterface;
use Interop\Container\ContainerInterface;
use ToolkitApi\Toolkit;

class ToolkitFactory implements FactoryInterface
{

    public function __invoke(ContainerInterface $container, $requestedName, array $options = null )
    {    
            /** @var \Zend\Db\Adapter\Adapter $databaseAdapter
            $databaseAdapter = $container->get('Zend\Db\Adapter\Adapter');

            $databaseConnection = $databaseAdapter->getDriver()->getConnection()->getResource();
     
            return new Toolkit($dbConn, null, null, 'pdo');
        }
    }
}
```

Here is the current documentation with [Toolkit Examples](https://docs.roguewave.com/en/zend/Zend-Server-7-IBMi/content/toolkit_sample_scripts.htm){:target="_blank" rel="noopener"}

## Need To Re-add the IBM i Repos

The IBM i repos live here: http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo

To re-add them if you bork something:

```
    yum-config-manager --add-repo http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo

```

This does take having the yum-config-manager already installed. If you do not have that already installed you might want to reach out to someone at IBM for help. 

Or you can use our guide to install the packages found here: [How To Setup Open Source Package Manager if you don't have SSH access](https://ospm.k3s.com)

# Potential Speed Improvements

There are areas of the ODBC jobs that can have impact on the speed and response of your application depending on what you have enabled / disabled or running from a log standpoint. A couple key areas to take a look at are:

## Prestart ODBC Jobs (QZDASOINIT)

Prestarting ODBC jobs can ensure there is enough jobs available and waiting when the requests come in, as well as making sure the jobs cycle through fast enough to keep things working. [IBM i Database Host Server and the QZDASOINIT Prestart Jobs](https://www.ibm.com/support/pages/ibm-i-database-host-server-and-qzdasoinit-prestart-jobs)

## Exit Programs

Exit Propgrams are a fantastic tool within the IBM i world that can attach a program to lots of individual actions within the system. As an example, a program can be called every time ODBC is used to access the database. This program can be used to check extra security or do a number of logging options. If this is forgetten and left on the ODBC connection, it can add a tremendous amount of overhead. Always double check your exit programs that they are needed and part of your application environment. [Harnessing Your ODBC Users with Exit Programs](https://www.itjungle.com/2006/11/29/fhg112906-story02/)

## Trace TCP/IP Application (TRCTCPAPP)

You can enable tracing on a TCP/IP application to learn more about the traffic within your app. If this is left on there can be a significant amount of overhead. [Trace TCP/IP Application (TRCTCPAPP)](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_71/cl/trctcpapp.htm)

# About K3S (King III Solutions, Inc)

[K3S](https://k3s.com) is a software development company that specializes in inventory management and procurement solutions for the distribution industry. Their applications and solutions are developed to run on the IBM i OS (the best enterprise level OS!) and interface with any ERP application on any platform. 

As well K3S open sources many of its [Guides & Utilities ](https://technical.k3s.com/docs/utilities/) in an effort to improve the IBM i community. 
