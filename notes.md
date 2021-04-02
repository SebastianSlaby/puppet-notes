# Common commands
facter - facts about a system
agent has to request cert signage from the server
server has to sign cert
puppet describe resource
puppet resource resourcename ...
```puppet resource --types```  or ```puppet describe --list ```- lists all resource types available
# Agent-client 
Agent provides facts to the server
The server provides a catalog with the desired state of the agent
Agent returns facts to the server again after the changes
By default the agents runs checks in 30 minutes intervals, this can be changed in the config file of the agent

Each server is classified after authenticating, it determines what the state of that server should be. 
Classes are defined in module directories with the structure ./nameofclass/manifests
Classification can be done in 2 ways:
-Manifest file (site.pp)
 Use puppet config to determine it's location
```puppet config print manifest```
/etc/puppetlabs/code/environments/production/manifests
-ENC external node classifier:
 - Enterprise Console
 - Foreman


#File syntax:

## Defining classes
```
class classname {
  resource { 'name':
  property => value,
  prop2 => value,
  }
  resource2 { 'name':
  prop => value,
  }

}
```
## Assigning classes to nodes
```
node "vm1.test.net" {
  include sysadmins
}

```

# Other resources
## package
## service 
Status start stop etc can be overridden in case of non-standard services
### notify 
Pushes notification to log or stdout in case of puppet apply
## exec 
Execute any arbitrary command. Not idempotent by default, can be made idempotent by using creates, onlyif or unless attributes. Exec requires a full path to execute files.
```creates => '/path'``` - if the script run by exec creates a file/directory use creates. If this path exists, the exec will not run.
```unless => 'test -d /opt/app'``` - run the exec unless the command specified returns 0.
```onlyif => 'test -d /opt.app'``` - run the exec only if the command specified returns 0.

# Namevar
Names of the resources must be unique per catalog. An example with the package resource type:
```
package { 'httpd':
 ensure => installed,
}
package { 'apache':
 ensure => installed,
 name => 'httpd',
}
```
In the 1st example, the name is httpd and the installed package is httpd too. In the 2nd example. the name of the resource is apache, but the installed package is httpd.
Package also has the provider var, it can be used to use, for example, pip instead of yum

# Managing files
Use the ```file``` resource type to mange files. ensure can be set to file, symlink or dir. The owner. group and mode can be specified too.
To manage a file's content use ```source``` or ```content```.  Source specifies a location on the puppet server to serve the file statically, whereas content specifies a string value to populate the file.
The ```source``` attribute takes and URI argument which looks like ```puppet:///modules/apache/httpd.conf``` (```puppet://hostname/mountpoint/path```). Mountpoints are entries in ```fileserver.conf```. The ```modules``` mountpoint is a special internal mountpoint. The server wil search in a predetermined location in the ```modulepath```. The hostname can be ```/``` when refering to the puppet server. In the example above a module in one of the directories specifeid in ```modulepath``` has to be created with a file structure like:
```
apache
|- files
   |- httpd.conf
|- manifests
   | - init.pp
```
Use the ```replace => false``` argument to make puppet not replace the content in case the file exists. The permissions and owners are still changed.
