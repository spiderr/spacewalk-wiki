# Registering Clients



Instructions for registering client systems you wish to manage with Spacewalk 1.3

Spacewalk 1.2 instructions are available at [[RegisteringClients12]].
## Before Starting

  1. Create a base channel within Spacewalk (Channels > Manage Software Channels > Create New Channel)

  2. Create an activation key with the new base channel. When creating a registration key do not use the generate function, create a human-readable version. eg: fedora-server-channel. This makes your installation more understandable and provides greater logical consistency to the whole system. On the other hand, if you want to prevent people from getting access to your channels, letting Spacewalk to generate random activation key name is the best way to go.
### Note about rhnreg_ks --force



rhnreg_ks is used for registration of clients to Spacewalk. If you need to re-register a client to your Spacewalk server or change registration from one environment or server to another Spacewalk server then use the "--force" flag with rhnreg_ks, otherwise there is no need to use "--force".
## Fedora



 1. Install the Spacewalk client yum repository
    * For Fedora 13:

    # rpm -Uvh http://spacewalk.redhat.com/yum/1.3/Fedora/13/i386/spacewalk-client-repo-1.3-1.fc13.noarch.rpm
    * For Fedora 14:

    # rpm -Uvh http://spacewalk.redhat.com/yum/1.3/Fedora/14/i386/spacewalk-client-repo-1.3-1.fc14.noarch.rpm
 2. Install client packages
   
    # yum install rhn-client-tools rhn-check rhn-setup rhnsd m2crypto yum-rhn-plugin
       }}}
     3. Register your Fedora system to Spacewalk using the activation key you created earlier
       {{{
    # rhnreg_ks --serverUrl=http://YourSpacewalk.example.org/XMLRPC --activationkey=<key-with-fedora-custom-channel> 
       }}}
## CentOS 5 or Red Hat Enterprise Linux 5 and 6

    

    '''Warning:''' If you are installing these packages on a Red Hat Enterprise Linux installation it will override some of the original base packages and you may well be invalidating your support agreement with Red Hat!
    
      1. Install the Spacewalk yum repository
     * RHEL5 /CentOS 5
       {{{
    # rpm -Uvh http://spacewalk.redhat.com/yum/1.3/RHEL/5/i386/spacewalk-client-repo-1.3-1.el5.noarch.rpm
       }}}
     * RHEL6
       {{{
    # rpm -Uvh http://spacewalk.redhat.com/yum/1.3/RHEL/6/i386/spacewalk-client-repo-1.3-1.el6.noarch.rpm
       }}}
      2. The latest client tools bring the upstream development to your client boxes. That means that the packages may have dependencies that are not found in core Red Hat Enterprise Linux. These dependencies can be found in EPEL, just like for the Spacewalk server:
     * EPEL5
       {{{
    # BASEARCH=$(uname -i)
    # rpm -Uvh http://download.fedora.redhat.com/pub/epel/5/$BASEARCH/epel-release-5-4.noarch.rpm
       }}}
     * EPEL6
    {{{
    # BASEARCH=$(uname -i)
    # rpm -Uvh http://download.fedora.redhat.com/pub/epel/6/$BASEARCH/epel-release-6-5.noarch.rpm
       }}}
      3. Install client packages
        {{{
    # yum install rhn-client-tools rhn-check rhn-setup rhnsd m2crypto yum-rhn-plugin
        }}}
      4. Register your CentOS or Red Hat Enterprise Linux system to Spacewalk using the activation key you created earlier
       {{{
    # rhnreg_ks --serverUrl=http://YourSpacewalk.example.org/XMLRPC --activationkey=<key-with-rhel-custom-channel> 
       }}}
## CentOS 4

    

    Registering a CentOS 4 server to Spacewalk is exactly the same as it would be for CentOS 5, but rhnreg_ks, rhn_check and other related scripts are located in the package '''up2date''', and not in '''rhn-setup'''.
    
      1. Enable '''spacewalk-tools''' repo for Yum and install '''up2date''' package:
    {{{
    # rpm -ivh http://stahnma.fedorapeople.org/spacewalk-tools/spacewalk-client-tools-0.0-1.noarch.rpm
    # yum install up2date 
  1. Register your CentOS system to Spacewalk using the activation key you created earlier:

    # rhnreg_ks --serverUrl=http://YourSpacewalk.example.org/XMLRPC --activationkey=<key-with-centos-custom-channel>
## Debian



Add this line to file /etc/apt/sources.list:

    deb http://miroslav.suchy.cz/spacewalk/debian spacewalk-unstable ./
*Note*: packages are not signed. We are working on direct inclusion in Debian and proper signing.

Register either using *rhnreg_ks* or *rhn_register*.

Run command:

    apt-get install apt-spacewalk
Then you need to add to file /etc/apt/sources.list

    deb spacewalk://<spacewalk_fqdn> channels: main child01 child02 
*Note*: you have to add there names of channel for now. In future versions it will not be required.
# Solaris 8, 9 or 10



See [Solaris Support](Solaris).