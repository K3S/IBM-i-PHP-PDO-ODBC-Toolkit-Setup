# Purpose

I have created this guide as a reference for myself, but also to help others who might be trying to accomplish the same thing. Lots of this info is copy and pasted and I do not represent it as my own, only that I have aggregated it into one place to make it easier to use (there were many different sources for all this). 

# Installing PHP RPM on IBM i

This is a guide on how to install the PHP RPMs from Zend. This is the community edition you can use as any normal open source software. The ODBC extension is not installed with the normal Zend Server version of PHP, so if you want to use ODBC with PHP you are going to need to install the RPMs. 

1.	Make sure the you have installed the Open Source Package Management (OSPM) from ACS [Getting started with Open Source Package Management in IBM i ACS](https://www.ibm.com/support/pages/getting-started-open-source-package-management-ibm-i-acs)

2.	From the OSPM install yum-utils from the Available Packages tab. This will allow you to add 3rd party packages. 

**NOTE: If you attempt to add the repo listed for PHP it will not find it. It is in a sub directory. As of now (12/12/2019) you want to use this command from the command line:

```
 yum-config-manager --add-repo http://repos.zend.com/ibmiphp/ppc64/
```

(3rd Party Open Source Repos for IBM i)[https://bitbucket.org/ibmi/opensource/src/master/docs/yum/3RD_PARTY_REPOS.md]

3.	As the repo RPMs for PHP were now added to ACS, now, just as you added yum-utils from the Available Packages tab, add the PHP packages / extensions you want. I would just add all of them that begin with php. They are not very large and you will end up probably using all of them.

4.	Configuring PHP: php is configured in this file `/QOpenSys/etc/php.ini`
Extensions are added via this directory: `/QOpenSys/etc/php/conf.d`

Default versions of configurations and all the extensions are added when you add the RPMs. 

Updates to RPMs: When there are updates shipped to the RPMs and the config files there is a good system to not get rid of your changes. If you are running the config files unchanged from install it will install the newer versions. But if you made changes to the config files it will keep the versions you have and write the newer default ones to the same directory with this naming scheme: _filename_.rpmnew


# Example ODBC Connection To DB2 On IBM i

These are some example connection strings and directions to help people connect via ODBC to DB2 on the IBM i. These examples include running a PHP application on a Linux / Windows server OR running PHP directly on the IBM i. 

## IBM Connection String Reference

[IBM ODBC Connection String Keywords](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_74/rzaik/connectkeywords.htm)

## Connecting A PHP Application Running On IBM i To A DB2 Database Running On IBM i 

This is an example of how to use PDO and ODBC to connect to DB2 on IBM i when PHP is running on the IBM i.

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

It is possible to call RPG from another server over ODBC. This uses your PDO connection referenced above. 

[IBM i PHP Toolkit](https://github.com/zendtech/IbmiToolkit)

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

