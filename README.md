# Purpose

I have created this guide as a reference for myself, but also to help others who might be trying to accomplish the same thing. 

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

# Installing PHP RPM on IBM i
I have created this guide as a reference for myself, but also to help others who might be trying to accomplish the same thing. 

These are some example connection strings and directions to help people connect via ODBC to DB2 on the IBM i. These examples include running a PHP application on a Linux / Windows server OR running PHP directly on the IBM i. 
