# MS Active Directory (AD) Connector
> ActiveDirectory/LDAP authentication backend extension for SAL with ldap group to business unit mapping.

This docker container extends [SAL](https://github.com/salopensource/sal) by adding a django authentication
backend with ActiveDirectory/LDAP integration.

Currently, this connector works with MS ActiveDirectory/LDAP only.

# Features
* Binding to the configured AD/LDAP server with username and password from the SAL/django user login.
* Creating a SAL/django user with field information from AD/LDAP (no password is stored in django!)
* Setting SAL user profile (GA, RW, RO) based on their AD/LDAP group.
* Assigning users to SAL business units based on their AD/LDAP group.
* Updating the user profile and business unit assignment at every login of the user.

# Docker Setup
The setup is idantical to a standard sal docker container as described in the [SAL Wiki](https://github.com/salopensource/sal/wiki/Docker).
In addition you have to link your `settings.py` into the container.

```bash
$ docker run -d --name="sal"\
  -p 80:8000 \
  --link postgres-sal:db \
  -e ADMIN_PASS=pass \
  -e DB_NAME=sal \
  -e DB_USER=admin \
  -e DB_PASS=password \
  --volume /path/to/your/settings.py:/home/docker/sal/sal/settings.py:ro
  macadmins/sal-activedirectory:3.2.14
```

# Settings
Following settings can/need to be configured in the django `settings.py` file to get this ActiveDirectory authentication backend to work.
See the `settings_example.py` for guidance.

## AUTHENTICATION_BACKENDS (important)
Make sure that the Active Directory authentication is configured as authentication backend.
```Python
AUTHENTICATION_BACKENDS = [
    'server.ADConnector.ADConnector',
    'django.contrib.auth.backends.ModelBackend',
]
```
If you don't intend to use the connector, just don't add it to your `AUTHENTICATION_BACKENDS`.

## AUTH_LDAP_SERVER_URI (mandatory)
URL of the AD/LDAP server.
```Python
AUTH_LDAP_SERVER_URI = 'ldaps://hostname.company.com:636'
```

## AUTH_LDAP_USER_DOMAIN
Domain of the AD/LDAP server.
```Python
AUTH_LDAP_USER_DOMAIN = company.com’
```
`username` will be converted to `username@company.com` (company.com = `AUTH_LDAP_USER_DOMAIN`) for the AD/LDAP authentication. The domain will only get appended to the username if the username does **not** end with the configured domain.

## AUTH_LDAP_USER_SEARCH (mandatory)
AD/LDAP search base for the user object.
```Python
AUTH_LDAP_USER_SEARCH = 'DC=department,DC=ads,DC=company,DC=com'
```

## AUTH_LDAP_USER_ATTR_MAP
Mapping of the AD/LDAP attributes to django attributes. If these settings are not configured, these default values are used.
```Python
AUTH_LDAP_USER_ATTR_MAP = {
    "username": "sAMAccountName",
    "first_name": "givenName",
    "last_name": "sn",
    "email": "mail"
}
```

## AUTH_LDAP_TRUST_ALL_CERTIFICATES
If you have a self signed certificate or an unknown certificate to the django server, you need to disable the certificate check by setting this value to `True`.
```Python
AUTH_LDAP_TRUST_ALL_CERTIFICATES = True
```
The parameter defaults to `False`, causing sal to trust certificates with a valide certificate chain only.

## AUTH_LDAP_USER_PREFIX
Django users that where created via AD/LDAP recieve the ldap_ preffix. (`username` becomes `ldap_username`). This allows to have local django users and AD/LDAP users in coexistence. Furthermore, local django users and AD/LDAP users are distinguished very easy.
```Python
AUTH_LDAP_USER_PREFIX = 'ldap_'
```

## AUTH_LDAP_USER_PROFILE (important)
Mapping of the user profile level (`GA` = Global Admin, `RW` = Read & Write, `RO` = Read Only, `SO` = Stats Only (*not implemented in SAL*)) to AD/LDAP groups. Mapping is a dictionary, where the key is the user profile level and the value corresponds to the AD/LDAP group. The group can be a single group or a list/tuple of AD/LDAP groups.
```Python
AUTH_LDAP_USER_PROFILE = {
                            'RO': ('CN=users-all,DC=department,DC=ad,DC=company,DC=com',),
                            'RW': ('CN=service-desk,DC=department,DC=ad,DC=company,DC=com',
                                   'CN=group-leader,OU=group,DC=department,DC=ad,DC=company,DC=com'),
                            'GA': 'CN=admin,DC=department,DC=ad,DC=company,DC=com',
                        }
```
The order of the user profile check is from `GA` to `RO` respectively `SO`. If a user is member of the `GA` **and** `RO` group, the assigned user profile level is `GA`.

## AUTH_LDAP_USER_TO_BUSINESS_UNIT (important)
Mapping of AD/LDAP groups to business units. Mapping is a dictionary, where the key is the name of the business unit and the value corresponds to the AD/LDAP group. The group can be a single group or a list/tuple of AD/LDAP groups.
```Python
AUTH_LDAP_USER_TO_BUSINESS_UNIT = {
                            '#ALL_BU':          ('CN=service-desk,DC=department,DC=ad,DC=company,DC=com',
                                                 'CN=group-leader,DC=department,DC=ad,DC=company,DC=com',),
                            'BusinessUnitU1':   ('CN=group-member,OU=group1,DC=department,DC=ad,DC=company,DC=com',),
                            'BusinessUnitU1':   ('CN=group-member,OU=group2,DC=department,DC=ad,DC=company,DC=com',),
                            'BusinessUnitU3':    'CN=group-member,OU=group3,DC=department,DC=ad,DC=company,DC=com',
                        }
```
**Attention**: `#ALL_BU` is a special business unit. All users in this configured groups get access to all existing business units.

# Logging
If something does not work as expected, an extensive debug logging can be turned on. This is implemented with the python logging module and can be configured in the django settings.
```Python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'standard': {
            'format': '%(asctime)-15s %(levelname)-7s %(filename)s:%(lineno)-4d: %(message)s',
            'datefmt': '%Y-%m-%d %H:%M:%S'
        },
    },
    'handlers': {
        'file': {
            'level': 'DEBUG',
            'class': 'logging.FileHandler',
            'filename': '/tmp/sal.log',
            'formatter': 'standard',
        },
    },
    'loggers': {
        'server': {
            'handlers': ['file'],
            'level': 'DEBUG',
            'propagate': True,
        },
    },
}
```
With this configuration everything is logged to `/tmp/sal.log`. Make sure that you turn off this option in production environment!

# FAQ
## Can existing django users with identical usernames coexist with new AD/LDAP users?
Yes: This can be accomplished with the setting `AUTH_LDAP_USER_PREFIX` very easily.
This configured prefix will be added to the django username, therefore existing django users with the same username as users in the AD/LDAP can still login.
## Is it possible to have a user with readonly rights in specific business unit and with write rights in a different one?
No: Unfortunately this is not possible by design of SAL. A user does always have **one** user profile which is valid for all assigned business unit.
## What happens if the authenticated user is not in any of the configured user profiles (GA, RW, RO)?
Then the default user profile is set, which is read only (RO). Defined at `UserProfile._meta.get_field('level').get_default()`.
## Can a user get assigned to all existing business units?
This is possible with business unit `#ALL_BU` in the `AUTH_LDAP_USER_TO_BUSINESS_UNIT` configuration.
## Assigned business units in SAL get reset every time a user logs in?
The user profile and all assigned business units of a user get a reset every time a user logs in.
Otherwise it is not possible to ensure that users get removed from business units they shouldn't have access anymore.
Therefore it is not possible and recommended to mix the assignment between SAL and the AD/LDAP configuration.
## Does this authentication work with another ldap implementation than AD/LDAP as well?
May be! But it is not tested or guaranteed. There are some AD/LDAP specific notations used to get nested groups (`memberOf:1.2.840.113556.1.4.1941:=`),
therefore I would not trust on a connection to another ldap implementation.
## This sounds like a nice feature but I don't have a need for it. How do I disable it?
The AD/LDAP connector is not enabled by default.  If you intend to use the connector, you haveto add it to your `AUTHENTICATION_BACKENDS` settings and configure it correctly.

# Possible improvements
There are always things which can be improved!
* **Bind with service account**. At the moment the AD/LDAP connection is initiated with the authenticated user. It could be that this user does not have the permissions to access the group memberships. Therefore AD/LDAP bind with a service account could be very useful.
* **Make it work with other ldap implementations than AD/LDAP**. At the moment, this authentication works with AD/LDAP only. May be someone can test and adapt it for other ldap implementation as well. Note: Take care of the nested groups.

