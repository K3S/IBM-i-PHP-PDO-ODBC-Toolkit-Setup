# Example ODBC Connection To DB2 On IBM i
These are some example connection strings and directions to help people connect via ODBC to DB2 on the IBM i. These examples include running a PHP application on a Linux / Windows server OR running PHP directly on the IBM i. 

## IBM Connection String Reference

[IBM ODBC Connection String Keywords](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_74/rzaik/connectkeywords.htm)


## Connecting A PHP Application Running On Linux / Windows To A DB2 Database Running On IBM i 

As PHP can run on multiple OSes, it can be beneficial in some circumstances to run PHP on another server and use the IBM i just for its DB2 database and Business Intelligence. 

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
