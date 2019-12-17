# Purpose

I have created this guide as a reference for myself, but also to help others who might be trying to accomplish the same thing. Lots of this info is copy and pasted and I do not represent it as my own, only that I have aggregated it into one place to make it easier to use (there were many different sources for all this). 

Notably thank you to Chuk Shirley, [Stephanie Rabinni](https://twitter.com/jordiwes){:target="_blank" rel="noopener"}, [Alan Seiden](https://twitter.com/alanseiden){:target="_blank" rel="noopener"}, Dave Dressler, and [Kevin Adler](https://twitter.com/kadler_ibm){:target="_blank" rel="noopener"}. 

# Installing PHP RPM on IBM i

This is a guide on how to install the PHP RPMs from Zend. This is the community edition you can use as any normal open source software. The ODBC extension is not installed with the normal Zend Server version of PHP, so if you want to use ODBC with PHP you are going to need to install the RPMs. Zend Server and PHP RPMs are pragmatically no different from a run time perspective. Most of the advantage in Zend Server is in the development, going through logging, and they have a custom package management. Otherwise most shops should just consider the PHP RPMs. 

1.	Setup Package Manager: Make sure the you have installed the Open Source Package Management (OSPM) from ACS [Getting started with Open Source Package Management in IBM i ACS](https://www.ibm.com/support/pages/getting-started-open-source-package-management-ibm-i-acs){:target="_blank" rel="noopener"}

2.	Install yum utilities: From the OSPM install yum-utils from the Available Packages tab. This will allow you to add 3rd party packages. 

3. Install PHP RPMs: From a shell command line (I recommend SSHing in to the server, but I believe you can use QSH) install the repo to PHP RPMs hosted by Zend. Here is a list of 3rd Party RPMs (make sure you read the note below before you add PHP)

   [3rd Party Open Source Repos for IBM i](https://bitbucket.org/ibmi/opensource/src/master/docs/yum/3RD_PARTY_REPOS.md){:target="_blank" rel="noopener"}

   **NOTE: If you attempt to add the repo listed for PHP it will not find it. It is in a sub directory. As of now (12/12/2019) you want to use this command from the command line:

```
 yum-config-manager --add-repo http://repos.zend.com/ibmiphp/ppc64/
```

   As the repo RPMs for PHP were now added to ACS, now, just as you added yum-utils from the Available Packages tab, add the PHP packages / extensions you want. I would just add all of them that begin with php. They are not very large and you will end up probably using all of them.

4.	Configuring PHP: php is configured in this file `/QOpenSys/etc/php.ini`
   Extensions are added via this directory: `/QOpenSys/etc/php/conf.d`

   Default versions of configurations and all the extensions are added when you add the RPMs. 

   Updates to RPMs: When there are updates shipped to the RPMs and the config files there is a good system to not get rid of your changes. If you are running the config files unchanged from install it will install the newer versions. But if you made changes to the config files it will keep the versions you have and write the newer default ones to the same directory with this naming scheme: _filename_.rpmnew

5. Create New Apache Instance On IBM i: 

6. Setup Aache To Run PHP:

   Once you have created the Apache, use the "Edit Configuration File" option on the
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

   Once the configration options are added, click the OK button at the bottom
of the page to save it.

7. Configure FastCGI: Now that Apache is set up for FastCGI, we need to configure a FastCGI handler
for application/x-httpd-php. Without this, the FastCGI processor won't work
and the PHP script will merely be downloaded by the web browser.

   Create a file called fastcgi.conf in `/www/<server name>/conf` (assuming the
defaul path for the webroot was chosen) with the following contents:

```
Server type="application/x-httpd-php" CommandLine="/QOpenSys/pkgs/bin/php-cgi" StartProcesses="1"
```

   You can now start the web server.

8. Test: Create a small index.php file in your webroot with the following code:

```
<?php phpinfo(); ?>
```

   And visit the virtual host you setup. You should see the PHP info page. If you do, you are running PHP via RPM on your IBM i. 

9. Recommended Additional Configuration Options For Apache, Speed, and a Common Issue: While your mileage may vary, I recommend these additional lines for your consideration in your Apache config to speed up your app. 

```
// These lines will add gzip compression to the data served. You want these near the top
LoadModule deflate_module /QSYS.LIB/QHTTPSVR.LIB/QZSRCORE.SRVPGM
AddOutputFilterByType DEFLATE application/x-httpd-php application/json text/css application/x-javascript application/javascript text/html

// This will turn on Keep Alive and allow users to reuse older connections, making the serving of data faster. 
TimeOut 30000
KeepAliveTimeout 30
HotBackup Off

// If your code looks funky or just run a lot of times it is your CCSID. This seems to help
DefaultFsCCSID 37
CGIJobCCSID 37
```

# Example ODBC Connection To DB2 On IBM i

These are some example connection strings and directions to help people connect via ODBC to DB2 on the IBM i. These examples include running a PHP application on a Linux / Windows server OR running PHP directly on the IBM i. 

## IBM Connection String Reference

[IBM ODBC Connection String Keywords](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_74/rzaik/connectkeywords.htm){:target="_blank" rel="noopener"}

## Connecting A PHP Application Running On IBM i To A DB2 Database Running On IBM i 

This is an example of how to use PDO and ODBC to connect to DB2 on IBM i when PHP is running on the IBM i. This will NOT work by default with Zend Server PHP. They do not include the ODBC extension needed. You either need to add the extension or run and install the PHP RPMs listed above. From a runtime perspective using ODBC there is no difference between the two Zend Server has some nice debugging tools and a set way to deploy applications, but otherwise is unneeded by most shops. 

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


## Connecting A PHP Application Running On Linux / Windows To A DB2 Database Running On IBM i 

As PHP can run on multiple OSes, it can be beneficial in some circumstances to run PHP on another server and use the IBM i just for its DB2 database and Business Intelligence (for example, calling RPG via the PHP Toolkit). 

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

## Calling RPG via The PHP Toolkit Over PDO ODBC When PHP Runs On Linux / Windows

It is possible to call RPG from another server over ODBC. This uses your PDO connection referenced above. Below is the code from my application running in Zend Framework. 

[IBM i PHP Toolkit](https://github.com/zendtech/IbmiToolkit){:target="_blank" rel="noopener"}

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

            $dbAdapter = $container->get('ToolkitAdapter');

            $dbConn = $dbAdapter->getDriver()->getConnection()->getResource();
     
            $tkConn = new Toolkit($dbConn, null, null, 'pdo');

            $tkConn->setOptions(array('stateless' => true, 'XMLServiceLib' => 'QXMLSERV'));

            return $tkConn;
        }

    }
}
```

Example calling code:
```
$value = 'test';
$numvalue = 100;
$param[] = $toolkit->AddParameterChar('BOTH', 10, 'Example Alpha of 10, null submit', 'PARM1', null);
$param[] = $toolkit->AddParameterChar('BOTH', 10, 'Example Alpha of 100, $value submit', 'PARM2', $K3SOBJ);
$param[] = $this->toolkit->AddParameterPackDec('BOTH', 5, 0, 'Example of Packed Dec 5 digits, 0 Decimals', 'PARM3', $numvalue);
$param[] = $this->toolkit->AddParameterPackDec('BOTH', 7, 2, 'Example of Packed Dec 7 digits, 2 Decimals', 'PARM4', null);

$result = $toolkit->PgmCall("RPGPROG", 'LIBNAME', $param, null, null);

// The $result will store the results from the Toolkit Call, can be referenced as an array
$returnvalue = $result['io_param']['PARM1'];
echo $returnvalue;
```

