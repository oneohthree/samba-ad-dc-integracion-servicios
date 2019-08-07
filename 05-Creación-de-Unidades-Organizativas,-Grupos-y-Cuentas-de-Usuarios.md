# Creación de Unidades Organizativas, Grupos y Cuentas de Usuarios

Las unidades organizativas son subdivisiones jerárquicas que agrupan entidades, tales como otras OUs, cuentas y grupos de usuarios, y ordenadores; facilitando la aplicación de políticas de grupos en un AD DC.

## Creación de Unidades Organizativas (Organizational Units - OU)

* Crear nueva Unidad Organizativa.

`samba-tool ou create 'OU=ACME,DC=example,DC=tld' --description='EXAMPLE.TLD Main Organizational Unit'`

## Creación de Grupos

* Crear nuevo Grupo de Usuarios perteneciente a la OU `ACME`.

`samba-tool group add IT --groupou='OU=ACME' --description='IT Technical Support Group'`

## Creación de Cuentas de Usuarios

Crear nueva Cuenta de Usuario perteneciente a la OU `ACME`.

    samba-tool user create 'john.doe' 'P@s$w0rd.456'` \
        --userou='OU=ACME' \
        --surname='Doe' \
        --given-name='John' \
        --initials='JD' \
        --job-title='Network Administrator' \
        --department='IT' \
        --company='EXAMPLE' \
        --description='IT Technical Support Account' \
        --mail='john.doe@example.tld'

* Añadir nuevo usuario al grupo `IT`.

`samba-tool group addmembers 'IT' john.doe`

* Añadir nuevo usuario a los grupos administrativos del AD DC.

`samba-tool group addmembers 'Administrators' john.doe`  
`samba-tool group addmembers 'Domain Admins' john.doe`  
`samba-tool group addmembers 'Schema Admins' john.doe`  
`samba-tool group addmembers 'Enterprise Admins' john.doe`  
`samba-tool group addmembers 'Group Policy Creator Owners' john.doe`

## Creación de Políticas de Grupos (Group Policy Object - GPO)

En los sistemas operativos Windows, una Política de Grupo (Group Policy Object - GPO) es un conjunto de configuraciones que define cómo debe lucir y comportarse el sistema para usuarios y/ó grupos de usuarios y ordenadores, previamente definidos y agrupados en OUs.

* Crear Política de Grupo para contraseñas

`samba-tool gpo create 'ACME Domain Password Policy' -U 'administrator'%'P@s$w0rd.123'`

* Vincular política creada a Unidad Organizativa

`samba-tool gpo setlink 'OU=ACME,DC=example,DC=tld' {D8D44688-6D73-4850-8373-2B2B7294483A} -U 'administrator'%'P@s$w0rd.123'`

* Comprobaciones

`samba-tool gpo listall`  
`samba-tool gpo listcontainers {D8D44688-6D73-4850-8373-2B2B7294483A} -U 'administrator'%'P@s$w0rd.123'`  
`samba-tool gpo getlink 'OU=ACME,DC=example,DC=tld' -U 'administrator'%'P@s$w0rd.123'`

**NOTA**: La modificación de los parámetros de las Políticas de Grupo se debe realizar mediante la aplicación gráfica `Group Policy Management` disponible en el paquete de herramientas administrativas `RSAT`.
