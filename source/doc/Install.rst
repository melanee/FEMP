#######################################
FreeBSD Nginx MySQL PHP  Opencart 3
#######################################

.. _Home:

-------------------------------
Installing FreeBSD Release 12.1
-------------------------------

`FreeBSD Handbook <https://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/index.html>`_

Downloads
===============================
Get FreeBSD AMD64::
    
    curl -O https://download.freebsd.org/ftp/releases/amd64/amd64/ISO-IMAGES/12.1/


Copy on USB::

    dd if=<iso image> of=/dev/<USB device> BS=4M status=progress


When installing also check box Ports and src.


Configuration
===============================

Hardware : MSI-H55E33 
-------------------------------

Graphics
-------------------------------
5.4.3. Kernel Mode Setting (KMS).
Add this line to /boot/loader.conf::

    kern.vty=vt

2D and 3D acceleration is supported on most Intel KMS driver graphics cards provided by Intel. It is now possible to use graphics drivers provided by the Ports framework or as packages.

* Use the legacy drivers available from graphics/drm-kmod. Driver name: i915kmsd.

::

    # pkg install -y drm-legacy-kmod
    

Initial rc config &  Interfaces 
-------------------------------
Ethernet 1Gig
Ethernet 10M
PPPoE
::

    #/etc/rc
    clear_tmp_enable="YES"
    kld_list="/boot/modules/i915kms.ko"
    sendmail_enable="NONE"
    # Set dumpdev to "AUTO" to enable crash dumps, "NO" to disable
    dumpdev="AUTO"
    zfs_enable="YES"
    hostname="mail.oogets.email"
    network_interfaces="lo0 tun0"
    ifconfig_tun0=
    cloned_interfaces="lo1"
    #ifconfig_re0="DHCP"
    ifconfig_re0_ipv6="inet6 accept_rtadv"
    ifconfig_xl0="inet 10.1.1.53 netmask 255.255.255.0"
    # PPPoE
    ppp_enable="YES"
    ppp_mode="ddial"
    #ppp_nat="YES"
    ppp_profile="bell"
    local_unbound_enable="YES"
    ntpdate_enable="YES"
    ntpd_enable="YES"

    # Services
    sshd_enable="YES"
    named_enable="YES"
    nginx_enable="yes"
    mysql_enable="yes"
    php_fpm_enable="YES"

 


Firewall
-------------------------------
`Enabling PF <https://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/firewalls-pf.html>`_ To use PF, its kernel module must be first loaded, by adding pf_enable=yes to /etc/rc.conf::
    
   # Enabling PF
   pf_enable=yes
   pf_flags=""
   pf_rules="/usr/local/etc/pf.conf"
   pflog_enable=yes
   pflog_logfile="/var/log/pflog" 
   pflog_flags=""


If there is a LAN behind the firewall and packets need to be forwarded for the computers on the LAN, or NAT is required, enable the following option::

   # sysrc gateway_enable=yes
   # sysctl net.inet.ip.forwarding=1

Check /etc/pf.conf for errors, but do not load ruleset::

    pfctl -vnf /usr/local/etc/pf.conf

PF rule set in /usr/local/etc/pf.conf::

    # /usr/local/etc/pf.conf
    # use tun interface to connect to PPPoE ext_if
    ext_phy = "re0"
    ext_if = "tun0"
    int_phy = "xl0"
    int_if = $int_phy


    ## Set and drop these IP ranges on public interface ##
    martians = "{ 127.0.0.0/8, 192.168.0.0/16, 172.16.0.0/12, \
              10.0.0.0/8, 169.254.0.0/16, 192.0.2.0/24, \
              0.0.0.0/8, 240.0.0.0/4 }"

    ## Set http(80)/https (443) port here ##
    webports = "{http, https}"

    ## enable these services ##
    int_tcp_services = "{domain, ntp, smtp, www, https, ftp, ssh}"
    int_udp_services = "{domain, ntp}"

    ## Skip loop back interface - Skip all PF processing on interface ##
    #set skip on lo

    ## Sets the interface for which PF should gather statistics such as bytes in/out and packets passed/blocked ##
    set loginterface $ext_if

    ## Set default policy ##
    block return in log all
    block out all


    # Drop all Non-Routable Addresses
    block drop in quick on $ext_if from $martians to any
    block drop out quick on $ext_if from any to $martians

    ## Blocking spoofed packets
    antispoof quick for $ext_if

    # Deal with attacks based on incorrect handling of packet fragments
    #scrub in all

    # Open SSH port which is listening on port 22 from VPN 139.xx.yy.zz Ip only
    # I do not allow or accept ssh traffic from ALL for security reasons
    #pass in quick on $ext_if inet proto tcp from 139.xxx.yyy.zzz to $ext_if_ip port = ssh flags S/SA keep state label "USER_RULE: Allow SSH from 139.xxx.yyy.zzz"
    ## Use the following rule to enable ssh for ALL users from any IP address #
    pass in inet proto tcp to $ext_if port ssh
    ### [ OR ] ###
    ## pass in inet proto tcp to $ext_if port 22

    # Allow Ping-Pong stuff. Be a good sysadmin
    pass inet proto icmp icmp-type echoreq

    # All access to our Nginx/Apache/Lighttpd Webserver ports
    pass proto tcp from any to $ext_if port $webports

    # Allow essential outgoing traffic
    pass out quick on $ext_if proto tcp to any port $int_tcp_services
    pass out quick on $ext_if proto udp to any port $int_udp_services

`PPPoE <https://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/userppp.html>`_ Config::

    #################################################################
    # /etc/ppp/ppp.conf
    # PPP Configuration File
    # Originally written by Toshiharu OHNO
    # Simplified 5/14/1999 by wself@cdrom.com
    #
    # See /usr/share/examples/ppp/ for some examples
    #
    # $FreeBSD: releng/12.1/usr.sbin/ppp/ppp.conf 338590 2018-09-11 17:05:26Z trasz $
    #################################################################

    default:
     set log Phase Chat LCP IPCP CCP tun command
     set ifaddr 10.0.0.1/0 10.0.0.2/0 255.255.255.0 0.0.0.0

    bell:
        set device PPPoE:re0
        set authname b1rhub72
        set authkey  Bell01
        set dial
        set login
        add default HISADDR                 # Add a (sticky) default route

    
It is important that the routed daemon is not started, as routed tends to delete the default routing table entries created by ppp::

    # sysrc router_enable="NO"

Nginx 1.4
-------------------------------

MySQL 8
-------------------------------

PHP 7.2
_______________________________


Opencart 3
-------------------------------
