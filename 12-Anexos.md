# Anexos

## Fichero de configuración prinicipal Squid+Samba AD DC

`cat /etc/squid/squid.conf`

    # WELCOME TO SQUID 3.5.x
    # ------------------------

    # OPTIONS FOR AUTHENTICATION
    # ---------------------------------------------------------------------
    # Kerberos authentication
    auth_param negotiate program /usr/lib/squid/negotiate_kerberos_auth -r -s GSS_C_NO_NAME
    auth_param negotiate children 20 startup=0 idle=1
    auth_param negotiate keep_alive off

    # NTLM authentication (fallback)
    auth_param ntlm program /usr/lib/squid/wsauth --dc1addr=dc.example.tld --dc1port=389
    auth_param ntlm children 20 startup=5 idle=10
    auth_param ntlm keep_alive off

    # Basic LDAP authentication (fallback)
    auth_param basic program /usr/lib/squid/basic_ldap_auth -R -b "dc=example,dc=tld" -D squid3@example.tld -w "P@s$w0rd.789" -f (|(userPrincipalName=%s)(sAMAccountName=%s)) -h dc.example.tld
    auth_param basic children 10
    auth_param basic realm PROXY.EXAMPLE.TLD
    auth_param basic credentialsttl 8 hours

    # NETWORK OPTIONS
    # ---------------------------------------------------------------------
    http_port 3128
    visible_hostname proxy.example.tld

    # OPTIONS WHICH AFFECT THE NEIGHBOR SELECTION ALGORITHM
    # ---------------------------------------------------------------------
    cache_peer proxy.tld parent 3128 0 no-query no-digest default
    cache_peer_domain proxy.tld

    # ACCESS CONTROL LISTS
    # ---------------------------------------------------------------------
    acl localnet src 192.168.0.0/24
    acl proxy_ip src 192.168.0.2/32
    acl local_domain dstdomain .example.tld
    acl porn_site dstdomain .porn.com
    acl CUBA dstdomain .cu
    acl AUTH proxy_auth REQUIRED

    # Kerberos group mapping
    external_acl_type INTRANET ttl=300 negative_ttl=60 %LOGIN /usr/lib/squid/ext_kerberos_ldap_group_acl -a -g Intranet -D EXAMPLE.TLD
    external_acl_type INTERNET ttl=300 negative_ttl=60 %LOGIN /usr/lib/squid/ext_kerberos_ldap_group_acl -a -g Internet -D EXAMPLE.TLD
    external_acl_type UNRESTRICTED ttl=300 negative_ttl=60 %LOGIN /usr/lib/squid/ext_kerberos_ldap_group_acl -a -g Unrestricted -D EXAMPLE.TLD
    acl intranet external INTRANET
    acl internet external INTERNET
    acl unrestricted external UNRESTRICTED

    # LDAP group mapping
    external_acl_type memberof %LOGIN /usr/lib/squid/ext_ldap_group_acl -R -K -S -b "DC=example,DC=tld" -D squid3@example.tld -w "P@s$w0rd.789" -f "(&(objectClass=person)(sAMAccountName=%v)(memberof=cn=%g,OU=Proxy,OU=ACME,DC=example,DC=tld))" -h dc.example.tld
    acl LDAPintranet external memberof Intranet
    acl LDAPinternet external memberof Internet
    acl LDAPunrestricted external memberof Unrestricted

    acl SSL_ports port 443
    acl Safe_ports port 80 21 443 70 210 280 488 591 777 1025-65535
    acl PURGE method PURGE
    acl CONNECT method CONNECT

    # HTTP_ACCESS
    # ---------------------------------------------------------------------
    http_access allow localhost manager
    http_access allow proxy_ip manager
    http_access deny manager

    http_access deny !Safe_ports
    http_access deny CONNECT !SSL_ports
    http_access allow PURGE localhost
    http_access deny PURGE
    http_access allow localhost

    http_access deny !AUTH
    # Using Kerberos
    http_access allow localnet unrestricted
    http_access allow localnet internet !porn_site
    http_access allow localnet intranet CUBA
    # Using basic LDAP
    http_access allow localnet LDAPunrestricted
    http_access allow localnet LDAPinternet !porn_site
    http_access allow localnet LDAPintranet CUBA
    http_access deny all

    # MEMORY CACHE OPTIONS
    # ---------------------------------------------------------------------
    cache_mem 256 MB
    maximum_object_size_in_memory 1024 KB
    memory_replacement_policy heap GDSF

    # DISK CACHE OPTIONS
    # ---------------------------------------------------------------------
    cache_replacement_policy heap LFUDA
    cache_dir aufs /var/spool/squid 1024 16 256
    minimum_object_size 128 KB
    maximum_object_size 5 MB
    cache_swap_low 90
    cache_swap_high 100
    offline_mode off

    # LOGFILE OPTIONS
    # ---------------------------------------------------------------------
    logfile_rotate 0
    logformat squid %ts.%03tu %6tr %>a %Ss/%03>Hs %<st %rm %ru %[un %Sh/%<a %mt
    access_log /var/log/squid/access.log squid
    cache_store_log /var/log/squid/store.log
    mime_table /usr/share/squid/mime.conf
    pid_filename /var/run/squid.pid

    # OPTIONS FOR TROUBLESHOOTING
    # ---------------------------------------------------------------------
    cache_log /var/log/squid/cache.log
    debug_options ALL,1
    coredump_dir /squid/var/cache/squid

    # OPTIONS FOR TUNING THE CACHE
    # ---------------------------------------------------------------------
    refresh_pattern ^ftp:           1440    20%     10080
    refresh_pattern ^gopher:        1440    0%      1440
    refresh_pattern -i (/cgi-bin/|\?) 0     0%      0
    refresh_pattern .               0       20%     4320

    # INTERNAL ICON OPTIONS
    # ---------------------------------------------------------------------
    icon_directory /usr/share/squid/icons
    global_internal_static on
    short_icon_urls on

    # ERROR PAGE OPTIONS
    # ---------------------------------------------------------------------
    error_directory /usr/share/squid/errors/Spanish
    err_page_stylesheet /etc/squid/errorpage.css

    # OPTIONS INFLUENCING REQUEST FORWARDING
    # ---------------------------------------------------------------------
    always_direct allow local_domain
    always_direct deny all
    never_direct allow all

    # DNS OPTIONS
    # ---------------------------------------------------------------------
    hosts_file /etc/hosts
    dns_v4_first on
    dns_timeout 1 seconds
    dns_nameservers proxy.tld

    # MISCELLANEOUS
    # ---------------------------------------------------------------------
    forwarded_for on
    cachemgr_passwd MyS3cr3tP@s$w0rd all

## Fichero de configuración prinicipal eJabberd+Samba AD DC

`cat /etc/ejabbed/ejabberd.yml`

    ###'  ejabberd configuration file

    ###.  =======
    ###'  LOGGING
    loglevel: 4
    log_rotate_size: 0
    log_rotate_date: ""
    log_rate_limit: 100

    ###.  ================
    ###'  SERVED HOSTNAMES
    hosts:
      - "example.tld"

    ###.  ===============
    ###'  LISTENING PORTS
    listen:
      -
        port: 5222
        ip: "::"
        module: ejabberd_c2s
        certfile: "/etc/ejabberd/ejabberd.pem"
        starttls: true
        protocol_options:
          - "no_sslv3"
        max_stanza_size: 65536
        shaper: c2s_shaper
        access: c2s
        zlib: true
        resend_on_timeout: if_offline
      -
        port: 5269
        ip: "::"
        module: ejabberd_s2s_in
      -
        port: 5280
        ip: "::"
        module: ejabberd_http
        request_handlers:
          "/websocket": ejabberd_http_ws
        web_admin: true
        http_bind: true
        tls: true
        certfile: "/etc/ejabberd/ejabberd.pem"
    disable_sasl_mechanisms: "digest-md5"

    ###.  ==================
    ###'  S2S GLOBAL OPTIONS
    s2s_use_starttls: optional
    s2s_certfile: "/etc/ejabberd/ejabberd.pem"
    s2s_protocol_options:
      - "no_sslv3"
    outgoing_s2s_families:
       - ipv4

    ###.  ==============
    ###'  AUTHENTICATION
    auth_password_format: scram
    fqdn: "jb.example.tld"
    auth_method: ldap
    ldap_servers: "dc.example.tld"
    ldap_encrypt: none
    ldap_port: 389
    ldap_rootdn: "ejabberd@example.tld"
    ldap_password: "P@s$w0rd.012"
    ldap_base: "OU=ACME,DC=example,DC=tld"
    ldap_uids: {"sAMAccountName": "%u"}
    ldap_filter: "(&(objectClass=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"

    ###.  ===============
    ###'  TRAFFIC SHAPERS
    shaper:
      normal: 1000
      fast: 50000
    max_fsm_queue: 1000

    ###.   ====================
    ###'   ACCESS CONTROL LISTS
    acl:
      admin:
        user:
          - "administrator@example.tld"
          - "john.doe@example.tld"
      local:
        user_regexp: ""
      loopback:
        ip:
          - "127.0.0.0/8"

    ###.  ============
    ###'  SHAPER RULES
    shaper_rules:
      max_user_sessions: 10
      max_user_offline_messages:
        - 5000: admin
        - 100
      c2s_shaper:
        - none: admin
        - normal
      s2s_shaper: fast

    ###.  ============
    ###'  ACCESS RULES
    access_rules:
      local:
        - allow: local
      c2s:
        - deny: blocked
        - allow
      announce:
        - allow: admin
      configure:
        - allow: admin
      muc_create:
        - allow: local
      pubsub_createnode:
        - allow: local
      register:
        - allow
      trusted_network:
        - allow: loopback

    ###.  ================
    ###'  DEFAULT LANGUAGE
    language: "es"

    ###.  =======
    ###'  MODULES
    modules:
      mod_adhoc: {}
      mod_admin_extra: {}
      mod_announce:
        access: announce
      mod_blocking: {}
      mod_caps: {}
      mod_carboncopy: {}
      mod_client_state: {}
      mod_configure: {}
      mod_disco: {}
      mod_echo: {}
      mod_http_bind: {}
      mod_last: {}
      mod_mam: {}
      mod_muc:
        host: "conference.@HOST@"
        access:
          - allow
        access_admin:
          - allow: admin
        access_create: muc_create
        access_persistent: muc_create
      mod_muc_admin: {}
      mod_offline:
        access_max_user_messages: max_user_offline_messages
      mod_ping: {}
      mod_privacy: {}
      mod_private: {}
      mod_pubsub:
        access_createnode: pubsub_createnode
        ignore_pep_from_offline: true
        last_item_cache: false
        plugins:
          - "flat"
          - "hometree"
          - "pep"
      mod_roster:
        versioning: true
      mod_shared_roster_ldap:
        ldap_base: "OU=ACME,DC=example,DC=tld"
        ldap_groupattr: "department"
        ldap_groupdesc: "department"
        ldap_memberattr: "sAMAccountName"
        ldap_useruid: "sAMAccountName"
        ldap_userdesc: "cn"
        ldap_rfilter: "(objectClass=user)"
        ldap_filter: "(&(objectClass=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"
      mod_stats: {}
      mod_time: {}
      mod_vcard_ldap:
        ldap_uids: {"sAMAccountName": "%u"}
        matches: infinity
        ldap_vcard_map:
          "NICKNAME": {"%s": ["givenName"]}
          "FN": {"%s": ["displayName"]}
          "EMAIL": {"%s": ["mail"]}
          "GIVEN": {"%s": ["givenName"]}
          "MIDDLE": {"%s": ["middleName"]}
          "FAMILY": {"%s": ["sn"]}
          "ORGNAME": {"%s": ["company"]}
          "ORGUNIT": {"%s": ["department"]}
          "TITLE": {"%s": ["title"]}
          "TEL": {"%s": ["telephoneNumber"]}
        ldap_search_fields:
          "User": "%u"
          "Full Name": "displayName"
          "Email": "mail"
        ldap_search_reported:
          "Full Name": "FN"
          "Nickname": "NICKNAME"
          "Email": "EMAIL"
      mod_version: {}
    allow_contrib_modules: true

## Ficheros de configuración prinicipal Postfix+Samba AD DC

`cat /etc/postfix/main.cf`

    ### MAIN
    mydomain = example.tld
    myhostname = mail.$mydomain
    myorigin = $mydomain
    mydestination = $myhostname, localhost.$mydomain, localhost
    smtpd_banner = $myhostname ESMTP
    biff = no
    append_dot_mydomain = no
    readme_directory = no
    compatibility_level = 2
    home_mailbox = Maildir/
    alias_maps = hash:/etc/aliases
    alias_database = hash:/etc/aliases
    relayhost =
    mynetworks = 127.0.0.0/8
    mailbox_size_limit = 0
    recipient_delimiter = +
    inet_interfaces = all
    inet_protocols = ipv4

    ### SASL
    smtpd_sasl_auth_enable = yes
    broken_sasl_auth_clients = yes

    ### TLS
    smtp_tls_security_level = may
    smtpd_tls_ask_ccert = yes
    smtpd_tls_security_level = may
    smtpd_tls_auth_only = no
    smtp_tls_note_starttls_offer = yes
    smtp_tls_loglevel = 1
    smtpd_tls_loglevel = 1
    smtpd_tls_received_header = yes
    smtpd_tls_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
    smtpd_tls_mandatory_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
    smtp_tls_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
    smtp_tls_mandatory_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
    lmtp_tls_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
    lmtp_tls_mandatory_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
    smtpd_tls_mandatory_ciphers = medium
    tls_medium_cipherlist = AES128+EECDH:AES128+EDH
    smtpd_tls_exclude_ciphers =
        aNULL,
        eNULL,
        EXPORT,
        DES,
        RC4,
        MD5,
        PSK,
        aECDH,
        EDH-DSS-DES-CBC3-SHA,
        EDH-RSA-DES-CDC3-SHA,
        KRB5-DE5, CBC3-SHA
    smtpd_tls_dh1024_param_file = /etc/ssl/dh2048.pem
    tls_random_source = dev:/dev/urandom
    smtpd_tls_cert_file=/etc/ssl/certs/exampleMail.crt
    smtpd_tls_key_file=/etc/ssl/private/exampleMail.key
    smtpd_tls_CAfile = /etc/ssl/certs/exampleMail.crt
    smtpd_use_tls=yes
    smtpd_tls_session_cache_database =
        btree:${data_directory}/smtpd_scache
    smtp_tls_session_cache_database =
        btree:${data_directory}/smtp_scache
    smtpd_sasl_local_domain = $mydomain

    ### VIRTUAL ENVIRONMENT
    virtual_minimum_uid = 5000
    virtual_uid_maps = static:5000
    virtual_gid_maps = static:5000
    virtual_mailbox_base = /var/vmail
    virtual_mailbox_domains = $mydomain

    ### DOVECOT INTEGRATION
    smtpd_sasl_type = dovecot
    smtpd_sasl_path = private/auth
    local_transport = virtual
    virtual_transport = lmtp:unix:private/dovecot-lmtp
    dovecot_destination_recipient_limit = 1

    ### SAMBA AD DC INTREGRATION
    virtual_mailbox_maps =
        proxy:ldap:/etc/postfix/virtual_mailbox_maps.cf
    virtual_alias_maps =
        proxy:ldap:/etc/postfix/virtual_list_maps.cf
        proxy:ldap:/etc/postfix/virtual_alias_maps.cf
    local_recipient_maps = $virtual_mailbox_maps

    ### SMTPD CLIENT RESTRICTIONS
    smtpd_client_restrictions =
        permit_mynetworks
        permit_sasl_authenticated

    ### SMTPD RELAY RESTRICTIONS
    smtpd_relay_restrictions =
        permit_mynetworks
        permit_sasl_authenticated
        reject_unauth_destination

    ### SMTPD UNLISTED RESTRICTIONS
    smtpd_reject_unlisted_recipient = yes
    smtpd_reject_unlisted_sender = yes

    ### SMTPD HELO RESTRICTIONS
    smtpd_helo_required = yes
    smtp_helo_name = $myhostname
    smtpd_helo_restrictions =
        permit_mynetworks
        reject_non_fqdn_helo_hostname
        reject_unknown_helo_hostname
        reject_invalid_helo_hostname
        reject_invalid_hostname
        permit_sasl_authenticated

    ### SMTPD SENDER RESTRICTIONS
    smtpd_sender_login_maps =
        proxy:ldap:/etc/postfix/virtual_sender_login_maps.cf
    smtpd_sender_restrictions =
        permit_mynetworks
        reject_sender_login_mismatch
        reject_unknown_sender_domain
        warn_if_reject reject_unverified_sender
        reject_non_fqdn_sender
        reject_unlisted_sender
        permit_sasl_authenticated
    unverified_sender_reject_reason = Address verification failed

    ### SMTPD RECIPIENT RESTRICTIONS
    smtpd_recipient_limit = 10
    smtpd_recipient_overshoot_limit = $smtpd_recipient_limit
    smtpd_recipient_restrictions =
        reject_unauth_pipelining
        reject_unknown_recipient_domain
        reject_non_fqdn_recipient
        reject_unlisted_recipient
        permit_mynetworks
        permit_sasl_authenticated
        reject_unauth_destination
        permit
    unverified_recipient_reject_reason = Address lookup failed

    ### SMTPD DATA RESTRICTIONS
    smtpd_data_restrictions =
        reject_unauth_pipelining

    ### BCC
    enable_original_recipient = no
    always_bcc = archive@example.tld

`cat /etc/postfix/master.cf`

    smtp      inet  n       -       y       -       -       smtpd
        -o smtpd_tls_cert_file=/etc/ssl/certs/exampleMail.crt
        -o smtpd_tls_key_file=/etc/ssl/private/exampleMail.key
    submission  inet    n   -   y   -   10  smtpd
        -o syslog_name=postfix/submission
        -o smtpd_tls_security_level=encrypt
        -o smtpd_sasl_auth_enable=yes
        -o smtpd_tls_cert_file=/etc/ssl/certs/exampleMail.crt
        -o smtpd_tls_key_file=/etc/ssl/private/exampleMail.key
        -o smtpd_client_restrictions=permit_sasl_authenticated,reject
    pickup    unix  n       -       y       60      1       pickup
    cleanup   unix  n       -       y       -       0       cleanup
    qmgr      unix  n       -       n       300     1       qmgr
    tlsmgr    unix  -       -       y       1000?   1       tlsmgr
    rewrite   unix  -       -       y       -       -       trivial-rewrite
    bounce    unix  -       -       y       -       0       bounce
    defer     unix  -       -       y       -       0       bounce
    trace     unix  -       -       y       -       0       bounce
    verify    unix  -       -       y       -       1       verify
    flush     unix  n       -       y       1000?   0       flush
    proxymap  unix  -       -       n       -       -       proxymap
    proxywrite unix -       -       n       -       1       proxymap
    smtp      unix  -       -       y       -       -       smtp
    relay     unix  -       -       y       -       -       smtp
    showq     unix  n       -       y       -       -       showq
    error     unix  -       -       y       -       -       error
    retry     unix  -       -       y       -       -       error
    discard   unix  -       -       y       -       -       discard
    local     unix  -       n       n       -       -       local
    virtual   unix  -       n       n       -       -       virtual
    lmtp      unix  -       -       y       -       -       lmtp
    anvil     unix  -       -       y       -       1       anvil
    scache    unix  -       -       y       -       1       scache
    maildrop  unix  -       n       n       -       -       pipe
      flags=DRhu user=vmail argv=/usr/bin/maildrop -d ${recipient}
    uucp      unix  -       n       n       -       -       pipe
      flags=Fqhu user=uucp argv=uux -r -n -z -a$sender - $nexthop!rmail ($recipient)
    ifmail    unix  -       n       n       -       -       pipe
      flags=F user=ftn argv=/usr/lib/ifmail/ifmail -r $nexthop ($recipient)
    bsmtp     unix  -       n       n       -       -       pipe
      flags=Fq. user=bsmtp argv=/usr/lib/bsmtp/bsmtp -t$nexthop -f$sender $recipient
    scalemail-backend unix  -   n   n   -   2   pipe
      flags=R user=scalemail argv=/usr/lib/scalemail/bin/scalemail-store ${nexthop} ${user} ${extension}
    mailman   unix  -       n       n       -       -       pipe
      flags=FR user=list argv=/usr/lib/mailman/bin/postfix-to-mailman.py
      ${nexthop} ${user}
    dovecot   unix  -       n       n       -       -       pipe
        flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/dovecot-lda -f ${sender} -d ${recipient}

## Fichero de configuración prinicipal Dovecot+Samba AD DC

`cat /etc/dovecot/dovecot.conf`

    auth_default_realm = example.tld
    auth_mechanisms = plain login
    disable_plaintext_auth = yes
    first_valid_uid = 5000
    last_valid_uid = 5000
    log_path = /var/log/dovecot.log
    log_timestamp = "%Y-%m-%d %H:%M:%S"
    login_greeting = "exampleMail IMAP/POP3 Server"
    mail_access_groups = vmail
    mail_uid = 5000
    mail_gid = 5000
    mail_location = maildir:/var/vmail/%d/%n/Maildir
    namespace inbox {
        inbox = yes
        location =
        list = yes
        prefix =
        mailbox Borradores {
            special_use = \Drafts
            auto = subscribe
        }
        mailbox Junk {
            special_use = \Junk
            auto = subscribe
        }
        mailbox Enviados {
            special_use = \Sent
            auto = subscribe
        }
        mailbox Papelera {
            special_use = \Trash
            auto = subscribe
        }
        mailbox Archivos {
            special_use = \Archive
            auto = subscribe
        }
    }
    ssl_protocols = !SSLv3 !TLSv1 !TLSv1.1
    ssl = required
    ssl_cert = </etc/ssl/certs/exampleMail.crt
    ssl_key = </etc/ssl/private/exampleMail.key
    ssl_cipher_list = AES128+EECDH:AES128+EDH
    ssl_prefer_server_ciphers = yes
    ssl_dh_parameters_length = 2048
    verbose_ssl = no
    protocols = imap pop3 lmtp
    service lmtp {
        unix_listener /var/spool/postfix/private/dovecot-lmtp {
            user = postfix
            group = postfix
            mode = 0660
        }
        user = vmail
    }
    service imap-login {
        inet_listener imaps {
            address = *
            port = 993
        }
    }
    service pop3-login {
        inet_listener pop3s {
            address = *
            port = 995
        }
    }
    service auth {
        unix_listener /var/spool/postfix/private/auth {
            user = postfix
            group = postfix
            mode = 0660
        }
        unix_listener auth-master {
            user = vmail
            group = vmail
            mode = 0660
        }
        unix_listener auth-userdb {
            user = vmail
            group = vmail
            mode = 0660
        }
    }
    service quota-warning {
        executable = script /usr/local/bin/quota-warning
        unix_listener quota-warning {
            user = vmail
        }
    }
    userdb {
        args = /etc/dovecot/dovecot-ldap.conf
        driver = ldap
    }
    passdb {
        args = /etc/dovecot/dovecot-ldap.conf
        driver = ldap
    }
    protocol lmtp {
        mail_plugins = quota
    }
    protocol lda {
        mail_plugins = quota
    }
    protocol imap {
        mail_plugins = quota imap_quota
    }
    protocol pop3 {
        mail_plugins = quota
    }
    plugin {
        quota = maildir:User quota
        quota_rule = *:storage=0:messages=1000
        quota_warning = storage=95%% quota-warning 95 %u
        quota_grace = 10%%
        quota_exceeded_message = ESTE BUZON HA EXCEDIDO LA CUOTA ASIGNADA
    }

## Fichero de configuración prinicipal Roundcube+Samba AD DC

`cat /opt/roundcube/config/config.inc.php

    <?php
    // Database
    $config['db_dsnw'] = 'DBDRIVER://DBUSER:DBUSERPASSWD@DBHOST/DBNAME';
    // IMAP
    $config['default_host'] = 'ssl://127.0.0.1';
    $config['default_port'] = 993;
    $config['imap_auth_type'] = 'LOGIN';
    $config['imap_delimiter'] = '/';
    $config['imap_conn_options'] = array(
        'ssl' => array(
            'verify_peer'       => true,
            'verify_peer_name'  => false,
            'cafile'            => '/etc/ssl/certs/exampleMail.crt',
            'allow_self_signed' => false,
            'ciphers'           => 'TLSv1+HIGH:!aNull:@STRENGTH',
            'peer_name'         => 'mail.example.tld',
        ),
    );
    $config['imap_vendor'] = 'dovecot';
    // SMTP (submission)
    $config['smtp_server'] = 'tls://127.0.0.1';
    $config['smtp_port'] = 587;
    $config['smtp_user'] = '%u';
    $config['smtp_pass'] = '%p';
    $config['smtp_auth_type'] = 'LOGIN';
    $config['smtp_conn_options'] = array(
        'ssl' => array(
            'verify_peer'       => true,
            'verify_peer_name'  => false,
            'cafile'            => '/etc/ssl/certs/exampleMail.crt',
            'allow_self_signed' => false,
            'ciphers'           => 'TLSv1+HIGH:!aNull:@STRENGTH',
            'peer_name'         => 'mail.example.tld',
        ),
    );
    // System & User preferences
    $config['mail_domain'] = 'example.net';
    $config['username_domain'] = 'example.net';
    $config['support_url'] = 'https://www.example.tld';
    $config['product_name'] = 'Roundcube Webmail';
    $config['des_key'] = 'rcmail-!24ByteDESkey*Str';
    $config['plugins'] = array(
        'archive',
        'zipdownload',
        'newmail_notifier',
    );
    $config['skin'] = 'larry';
    $config['cipher_method'] = 'AES-256-CBC';
    $config['force_https'] = true;
    $config['useragent'] = 'Roundcube Webmail';
    $config['quota_zero_as_unlimited'] = true;
    $config['message_show_email'] = true;
    $config['language'] = 'es_ES';
    $config['create_default_folders'] = true;
    $config['timezone'] = 'America/Havana';
    $config['date_format'] = 'd/m/Y';
    $config['time_format'] = 'g:i a';
    $config['identities_level'] = 3;
    $config['login_autocomplete'] = 0;
    $config['preview_pane'] = true;
    $config['preview_pane_mark_read'] = 1;
    $config['refresh_interval'] = 600;
    $config['message_sort_col'] = 'date';
    $config['message_sort_order'] = 'DESC';
    $config['list_cols'] = array(
        'flag',
        'attachment',
        'fromto',
        'subject',
        'status',
        'date',
        'size',
    );
    $config['addressbook_sort_col'] = 'firstname';
    $config['show_images'] = 1;
    $config['default_font_size'] = '12pt';
    $config['layout'] = 'desktop';
    $config['autocomplete_addressbooks'] = array(
        'sql',
        'global_ldap_abook'
    );
    // Samba AD DC Address Book
    $config['ldap_public']["global_ldap_abook"] = array(
        'name'              => 'Mailboxes',
        'hosts'             => array('dc.example.tld'),
        'port'              => 389,
        'use_tls'           => false,
        'ldap_version'      => '3',
        'network_timeout'   => 10,
        'user_specific'     => false,
        'base_dn'           => 'OU=ACME,DC=example,DC=tld',
        'bind_dn'           => 'postfix@example.tld',
        'bind_pass'         => 'P@s$w0rd.345',
        'writable'          => false,
        'search_fields'     => array(
            'mail',
            'cn',
            'sAMAccountName',
            'displayName',
            'sn',
            'givenName',
        ),
        'fieldmap' => array(
            'name'          => 'cn',
            'surname'       => 'sn',
            'firstname'     => 'givenName',
            'title'         => 'title',
            'email'         => 'mail:*',
            'phone:work'    => 'telephoneNumber',
            'phone:mobile'  => 'mobile',
            'phone:workfax' => 'facsimileTelephoneNumber',
            'street'        => 'street',
            'zipcode'       => 'postalCode',
            'locality'      => 'l',
            'department'    => 'department',
            'notes'         => 'description',
            'photo'         => 'jpegPhoto',
        ),
        'sort'          => 'cn',
        'scope'         => 'sub',
        'filter'        => '(&(|(objectclass=person))(!(mail=archive@example.tld))(!(userAccountControl:1.2.840.113556.1.4.803:=2)))',
        'fuzzy_search'  => true,
        'vlv'           => false,
        'sizelimit'     => '0',
        'timelimit'     => '0',
        'referrals'     => false,
        'group_filters' => array(
            'departments' => array(
                'name'    => 'Lists',
                'scope'   => 'sub',
                'base_dn' => 'OU=Email,OU=ACME,DC=example,DC=tld',
                'filter'  => '(objectClass=group)',
            ),
        ),
    );

## Ficheros de publicación web Roundcube

### Servidor Web Nginx.

`cat /etc/nginx/sites-enabled/roundcube`

    proxy_cache_path /tmp/cache keys_zone=cache:10m levels=1:2 inactive=600s max_size=100m;
    server {
        listen 80;
        listen 443 ssl http2;
        root /opt/roundcube;
        server_name webmail.example.tld;
        if ($scheme = http) {
            return 301 https://$server_name$request_uri;
        }
        ssl_certificate /etc/ssl/certs/exampleMail.crt;
        ssl_certificate_key /etc/ssl/private/exampleMail.key;
        ssl_dhparam /etc/ssl/dh2048.pem;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
        ssl_ecdh_curve secp384r1;
        ssl_session_cache shared:SSL:10m;
        ssl_session_tickets off;
        ssl_stapling_verify on;
        ssi on;
        resolver 127.0.0.1 valid=300s;
        resolver_timeout 5s;
        add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
        add_header X-Content-Type-Options nosniff;
        proxy_cache cache;
        proxy_cache_valid 200 1s;
        location ~ [^/]\.php(/|$) {
            fastcgi_split_path_info ^(.+?\.php)(/.*)$;
            if (!-f $document_root$fastcgi_script_name) {
                return 404;
            }
            fastcgi_param HTTP_PROXY "";
            fastcgi_pass unix:/var/run/php/php7.3-fpm.sock;
            include snippets/fastcgi-php.conf;
        }
        location ~ /\. {
            deny all;
        }
        location = /favicon.ico {
            log_not_found off;
            access_log off;
        }
        location = /robots.txt {
            access_log off;
            log_not_found off;
        }
        location / {
            index index.php;
            location ~ ^/favicon.ico$ {
                root /opt/roundcube/skins/larry/images;
                log_not_found off;
                access_log off;
                expires max;
            }
            location ~ ^/(bin|SQL|config|temp|logs)/ {
                 deny all;
            }
            location ~ ^/(README|INSTALL|LICENSE|CHANGELOG|UPGRADING)$ {
                deny all;
            }
            location ~ ^/(.+\.md)$ {
                deny all;
            }
            location ~ ^/program/resources/(.+\.pdf)$ {
                deny all;
                log_not_found off;
                access_log off;
            }
            location ~ ^/\. {
                deny all;
                access_log off;
                log_not_found off;
            }
        }
        access_log /var/log/nginx/roundcube_access.log;
        error_log /var/log/nginx/roundcube_error.log;
    }

### Servidor Web Apache

`cat /etc/apache2/sites-enabled/roundcube.conf`

    <VirtualHost *:80>
        RewriteEngine on
        RewriteCond %{HTTPS} =off
        RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [QSA,L,R=301]
    </VirtualHost>

    <IfModule mod_ssl.c>
        <VirtualHost *:443>
            ServerName webmail.example.tld
            ServerAdmin webmaster@example.tld
            DocumentRoot /opt/roundcube
            DirectoryIndex index.php
            ErrorLog ${APACHE_LOG_DIR}/roundcube_error.log
            CustomLog ${APACHE_LOG_DIR}/roundcube_access.log combined
            SSLEngine on
            SSLCertificateFile /etc/ssl/certs/exampleMail.crt
            SSLCertificateKeyFile /etc/ssl/private/exampleMail.key
            <FilesMatch "\.(cgi|shtml|phtml|php)$">
                SSLOptions +StdEnvVars
            </FilesMatch>
            <Directory /usr/lib/cgi-bin>
                SSLOptions +StdEnvVars
            </Directory>
            BrowserMatch "MSIE [2-6]" nokeepalive ssl-unclean-shutdown downgrade-1.0 force-response-1.0
            BrowserMatch "MSIE [17-9]" ssl-unclean-shutdown
            <Directory /opt/roundcube>
                Options +FollowSymlinks
                AllowOverride All
                Require all granted
                SetEnv HOME /opt/roundcube
                SetEnv HTTP_HOME /opt/roundcube
                <IfModule mod_dav.c>
                    Dav off
                </IfModule>
            </Directory>
            <Directory /opt/roundcube/program/resources>
                <FilesMatch "\.(pdf)$">
                    Require all denied
                </FilesMatch>
            </Directory>
        </VirtualHost>
    </IfModule>

13.- Última revisión.

    * 20190801

14.- TODO

    `samba-tool domain level raise \
        [--domain-level=2012_R2 --forest-level=2012_R2]`
    ERROR: Domain function level can't be higher than the lowest function level of a DC!
