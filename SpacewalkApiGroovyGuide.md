# The Groovy Coders’ Guide to the Red Hat Network Satellite/Spacewalk API

## Intended Audience




The document is intended for system administrators and/or Groovy coders who wish to make use of the RedHat RHN Satellite v5.1 Server API. There may well be better ways to code the examples below and it should be noted that they have not been thoroughly examined for security issues.  Additionally, there are a few complex examples to help give you a better idea of some of the real life uses of the RHN Satellite Server API.

This guide was modified from the SpacewalkApiPerlGuide and is intended for those individuals to whom merely thinking of PERL gives them a headache, much less actually having to use it.
## Groovy XMLRPC Module



In order for your custom code or scripts to access the Satellite API, you must include the Groovy XMLRPC module (http://groovy.codehaus.org/XMLRPC).  This provides the essential groovy.net.xmlrpc.* classes needed for a Groovy XMLRPC client proxy.
## API Introduction



Every Groovy script that accesses the Satellite API must include some variation on the following code:


    import groovy.net.xmlrpc.*
    
    // Set the host appropriately
    def host = "satellite.company.com"
    
    def url = "http://${host}/rpc/api"
    def client = new XMLRPCServerProxy(url)
    
    // Use the client to obtain the authentication key
    def key = client.auth.login("username", "password")
    
    assert client != null && key != null  : "Login to RHN Satellite failed!"
    
    rhn.client.api.getApiNamespaces(rhn.key).each { apiNamespace ->
        println apiNamespace
    }
    
    assert client.auth.logout(key) == 1 : "Failed logging out of Satellite"
## RHNSession Helper



The problem with the above approach is the username and password of a valid (and probably privileged) Satellite user is stored in clear text in each and every one of your scripts. To avoid this, I create the file /etc/satellite_api.conf (which a filemode of 0600 and owned by root:root) which contains:


    satellite.company.com user password

and use a home grown Groovy class called RHNSession. RHNSession consists of:


    import groovy.net.xmlrpc.*
    
    class RHNSession {
    
        def configFile = "/etc/satellite_api.conf"
    
        def client
        def key
    
        def isLoggedIn() {
            client != null && key != null
        }
    
        def login() {
            def config = new File(configFile)
            assert config.exists() : "Failed reading ${configFile} because it doesn't exist"
    
            def (hostname, username, password) = config.readLines()[0].split(" ")
    
            client = new XMLRPCServerProxy("http://${hostname}/rpc/api")
            key = client.auth.login(username, password)
    
            assert isLoggedIn() : "Failed logging into ${hostname} as ${username}"
        }
    
        def logout() {
            assert isLoggedIn() : "Not logged into Satellite"
            assert client.auth.logout(key) == 1 : "Failed logging out of Satellite"
            client = null
            key = null
        }
    
    }

So your minimal API Groovy script is now:


    RHNSession rhn = new RHNSession()
    rhn.login()
    
    rhn.client.api.getApiNamespaces(rhn.key).each { apiNamespace ->
        println apiNamespace
    }
    
    rhn.logout()
### Export Server Information




    RHNSession rhn = new RHNSession()
    // Step 1: Login!
    rhn.login()
    
    // Make sure we're logged in
    assert isLoggedIn() : "Failed logging in, check username and password"
    
    // Find all hosts and print them in csv format
    println "ID,name,hostname,last_checkin,ip,cpu-arch,cpu-count,memory,swap,kernel"
    rhn.client.system.listSystems(rhn.key) { system ->
        def network = rhn.client.system.getNetwork(rhn.key, system.id)
        def memory = rhn.client.system.getMemory(rhn.key, system.id)
        def kernel = rhn.client.system.getRunningKernel(rhn.key, system.id)
        def cpu = rhn.client.system.getCpu(rhn.key, system.id)
        if ( cpu.class == String.class) {
            println "**** ${system.name} doesn't have CPU info... needs investigation"
        } else {
            println "${system.id},${system.name},${system.last_checkin},${network.hostname},${network.ip},${cpu.arch},${cpu.count},${memory.ram},${memory.swap},${kernel}"
        }
    }
    
    rhn.logout()
### Cloning Software Channels



This example is to demonstrate real world release management, where patches and updates must be vetted or promoted via a predefined release lifecycle.

# Identify the source channels; typically these are the RedHat source channels.


    RHNSession rhn = new RHNSession()
    rhn.login()
    
    // this map identifies the source channel's label and the new channels label
    // [ sourceLabel: destinationLabel ]
    def releaseChannels = [
        'rhel-i386-es-4':"rhel4-32",
        'rhel-i386-server-5':"rhel5-32",
        'rhel-i386-server-6':"rhel6-32",
        'rhel-x86_64-es-4':"rhel4-64",
        'rhel-x86_64-server-5':"rhel5-64",
        'rhel-x86_64-server-6':"rhel6-64"
    ]
    
    // Get the date and format it for naming the new channels
    def now = new Date()
    def year = now.format("yyyy")
    def month = now.format("MMM").toLowerCase()
    
    releaseChannels.each { original, label ->
        def release = "fy$year-$month-$label"
        println "Working on $release ..."
    
        // Make sure we're logged in
        assert rhn.isLoggedIn() : "Not logged into RHN Satellite"
    
        def orig = rhn.client.channel.software.getDetails(key, original)
        def prev = null
    
        try {
            prev = rhn.client.channel.software.getDetails(key, release)
        } catch (all) { /* We can ignore this exception; it most likely means channel not found, which is good */ }
    
        if (prev) {
            // the new release already exists!  it must be deleted... women and children first!
            println " -- Finding child channels for ${prev.label}"
            rhn.client.channel.software.listChildren(key, release).each { child ->
                println "   -- Deleting child channel ${child.label}"
                assert rhn.client.channel.software.delete(key, child.label) == 1 : "Failed deleting ${child.label}"
                Thread.sleep(1000)
            }
            assert rhn.client.channel.software.delete(key, prev.label) == 1 : "Failed removing parent channel ${prev.label}"
            println " -- ${prev.label} successfully deleted"
            Thread.sleep(1000)
        }
    
        def params = [:]
    
        if (orig.name) params << [name: "FY $year ${month.toUpperCase()} - ${orig.name}"]
        params << [label: release]
        if (orig.summary) params << [summary: orig.summary]
        if (orig.parent_channel_label) params << [parent_label: release]
        if (orig.gpg_key_url) params << [gpg_url: orig.gpg_key_url]
        if (orig.gpg_key_id) params << [gpg_id: orig.gpg_key_id]
        if (orig.gpg_key_fp) params << [gpg_fingerprint: orig.gpg_key_fp]
        if (orig.description) params << [description: orig.description]
    
        println " ++ Cloning $original as $release"
        assert rhn.client.channel.software.clone(
                key, original, params, false
            ) > 0 : " ** Failed creating parent channel $release"
        Thread.sleep(1000)
    
        assert rhn.client.channel.software.setGloballySubscribable(key, release, true) == 1 : "Failed setting ${release} as a globally subscribable channel"
        assert rhn.client.channel.access.setOrgSharing(key, release, "public") == 1 : "Failed setting ${release} as a public channel"
        assert rhn.client.channel.software.regenerateYumCache(key, release) == 1 : "Failed regenerating YUM cache for ${release}"
        Thread.sleep(1000)
    
        rhn.client.channel.software.listChildren(key, original).each { child ->
            params.clear()
    
            def childRelease = "fy$year-$month-${child.label}"
            def childName = child.name
    
            if (child.name) params << [name: "FY $year ${month.toUpperCase()} - ${childName}"]
            params << [label: childRelease]
            if (child.summary) params << [summary: child.summary]
            if (child.parent_channel_label) params << [parent_label: release]
            if (child.gpg_key_url) params << [gpg_url: child.gpg_key_url]
            if (child.gpg_key_id) params << [gpg_id: child.gpg_key_id]
            if (child.gpg_key_fp) params << [gpg_fingerprint: child.gpg_key_fp]
            if (child.description) params << [description: child.description]
    
            println "   ++ Cloning ${child.label} as child $childRelease"
            assert rhn.client.channel.software.clone(
                    key, child.label, params, false
                ) > 0 : "  ** Failed creating child channel $childRelease"
            Thread.sleep(1000)
    
            assert rhn.client.channel.software.setGloballySubscribable(key, childRelease, true) == 1 : "Failed setting ${childRelease} as a globally subscribable channel"
            assert rhn.client.channel.access.setOrgSharing(key, childRelease, "public") == 1 : "Failed setting ${childRelease} as a public channel"
            assert rhn.client.channel.software.regenerateYumCache(key, childRelease) == 1 : "Failed regenerating YUM cache for ${childRelease}"
            Thread.sleep(1000)
        }
    
        println "Sleeping 5 seconds before next channel..."
        Thread.sleep(5000)
    }
    
    rhn.logout()
### Download Config Channel Files




    import groovy.xml.MarkupBuilder
    
    RHNSession rhn = new RHNSession()
    rhn.login()
    
    // Make sure we're logged in
    assert rhn.isLoggedIn() : "Failed logging in, check username and password"
    
    // Get the list of all configuration channels
    def channelList = rhn.client.configchannel.listGlobals(rhn.key)
    channelList.each { channel ->
        // set the configuration channel name
        def name = channel.label
        println "Working on channel $name"
    
        // set the base path
        def base = "configchannels/$name"
    
        // this is the easiest way to create it; I'm on a unix machine and I'd rather not mess with
        // coding this in groovy... we're just making sure the base directory exists
        "/bin/mkdir -p $base".execute()
    
        // list the files in the configuration channel
        try {
            def fileList = rhn.client.configchannel.listFiles(rhn.key, name)
    
            // create the dircetory metadata (dump of the file info)
            File fileListFile = new File("$base/files.metadata")
            def writer = fileListFile.newWriter()
            //def writer = new StringWriter()
    
            def xml = new MarkupBuilder(writer)
            xml.files() {
                fileList.each { file ->
                    delegate.file {
                        'type'(file.type)
                        'directory'(file.directory)
                        'path'(file.path)
                        'last_modified'(file.last_modified)
                    }
                }
            }
            writer.close()
    
            def files = fileList.findAll { it.type == "file" }
            def dirs = fileList.findAll { it.type == "directory" }
            def symlinks = fileList.findAll { it.type == "symlinks" }
    
            dirs.each { dir ->
                println "Creating directory $base/${dir.path}"
                "/bin/mkdir -p $base/${dir.path}".execute()
            }
    
            files.each { file ->
                println "Creating file ${file.path}"
                def bp = "/usr/bin/dirname ${file.path}".execute().in.text
                "/bin/mkdir -p $base/$bp".execute()
    
                def info = rhn.client.configchannel.lookupFileInfo(rhn.key, name, [file.path])
    
                String type = info.type[0]
                String path = info.path[0]
                String target_path = info.target_path[0]
                int revision = info.revision[0]
                Date creation = info.creation[0]
                Date modified = info.modified[0]
                String owner = info.owner[0]
                String group = info.group[0]
                int permissions = info.permissions[0]
                String permissions_mode = info.permissions_mode[0]
                String selinux_ctx = info.selinux_ctx[0]
                boolean binary = info.binary[0]
                String md5 = info.md5[0]
                String macrostartdelimiter = info."macro-start-delimiter"[0]
                String macroenddelimiter = info."macro-end-delimiter"[0]
    
                // create the metadata (dump of the file info)
                metadataFile = new File("${base}/${file.path}.metadata")
                writer = metadataFile.newWriter()
                //writer = new StringWriter()
    
                xml = new MarkupBuilder(writer)
                xml.metadata() {
    
                    delegate.'channel'(name)
                    delegate.'type'(type)
                    delegate.'path'(path)
                    delegate.'target_path'(target_path)
                    delegate.'revision'(revision)
                    delegate.'creation'(creation)
                    delegate.'modified'(modified)
                    delegate.'owner'(owner)
                    delegate.'group'(group)
                    delegate.'permissions'(permissions)
                    delegate.'permissions_mode'(permissions_mode)
                    delegate.'selinux_ctx'(selinux_ctx)
                    delegate.'binary'(binary)
                    delegate.'md5'(md5)
                    delegate.'macro-start-delimiter'(macrostartdelimiter)
                    delegate.'macro-end-delimiter'(macroenddelimiter)
    
                }
                writer.close()
    
                // create the real file (dump the contents)
                contents = new File("${base}/${file.path}")
                contents.write(info[0].contents)
    
            }
    
            symlinks.each { symlink ->
                println "Creating symlink ${symlink.path}"
                def bp = "/usr/bin/dirname ${symlink.path}".execute().in.text
                "/bin/mkdir -p $base/$bp".execute()
                "/bin/ln -s ${symlink.target_path} $base/${symlink.path}".execute()
            }
        } catch (Exception e) { println "ERROR: ${e.message}"; e.printStackTrace() }
    }
    
    rhn.logout()
### Upload Config Channel Files




    RHNSession rhn = new RHNSession()
    
    // set the base path
    def base = "configchannels"
    "/bin/mkdir -p $base".execute()
    
    // Step 1: Login!
    rhn.login()
    
    // Make sure we're logged in
    assert isLoggedIn() : "Failed logging in, check username and password"
    
    // We either specify the list; OR we assume that any directories in our base path are the channel names
    //def channels = ["config-channel-label"]
    
    def channels = []
    def channelList = new File(base).eachDir {
        channels << it.name
    }
    
    channels.each { channel ->
        assert rhn.client.configchannel.channelExists(rhn.key, channel) == 1 : "Configuration channel [$channel] does not exist in satellite!"
    
        new File(base, channel).traverse { file ->
            switch (file.name) {
                case ".DS_Store":
                case ~/.*\.metadata$/:
                case ~/.*\.svn.*$/:
                break
                
                default:
                    if ( !file.isDirectory() ) {
                        def metadata
    
                        def metadataName = file.path + ".metadata"
                        def metadataFile = new File(metadataName)
                        if ( metadataFile.exists() ) {
                            metadata = new XmlSlurper().parse(metadataFile)
                        }
    
                        if (metadata) {
                            def actualPath = file.path.toString().substring(base.size() + channel.size() + 1)
    
                            assert channel.equals(metadata.channel.toString()) : "Specified channel in repository [$channel] is different than metadata channel [${metadata.channel}]"
                            assert actualPath.equals(metadata.path.toString()) : "Derived actual path [$actualPath] is different than metadata path [${metadata.path}]!"
    
                            def contentsName = file.path
                            def contentsFile = new File(contentsName)
                            def contents = contentsFile.text
    
                            def result = rhn.client.configchannel.createOrUpdatePath(
                                rhn.key,
                                metadata.channel.toString(),
                                metadata.path.toString(),
                                metadata.type.toString().equals("directory"),
                                [
                                    contents: contents.bytes.encodeBase64().toString(),
                                    contents_enc64: true,
                                    owner: metadata.owner.toString(),
                                    group: metadata.group.toString(),
                                    permissions: metadata.permissions_mode.toString(),
                                    selinux_ctx: metadata.selinux_ctx.toString(),
                                    "macro-start-delimiter": metadata."macro-start-delimiter".toString(),
                                    "macro-end-delimiter": metadata."macro-end-delimiter".toString(),
                                    revision: metadata.revision.toInteger() + 1
                                ]
                                )
                        }
                    }
    
                break
            }
        }
    }
    
    rhn.logout()
## API Insights

### Troubleshooting




Like Satellite itself, the API is not trivial and can take some time to get your head around. It’s very difficult to understand why your code isn’t working as expected if:

* You can’t visualise the data you are working with
* You don’t know how the module you are using is designed to work
## Sources



* Groovy XMLRPC Module (http://groovy.codehaus.org/XMLRPC)
* RedHat Satellite mailing list (https://www.redhat.com/mailman/listinfo/rhn-satellite-users)
* Documentation on the Satellite Server: (https://satellite.example.com/rhn/apidoc/index.jsp)
