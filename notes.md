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
Done in environmentpath/name/manifests
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

# Relationships
## Order of resources
Resource are read in the order they are written
Puppet suppors 3 ordering options(set in puppet.conf):
 - manifest - read in order, default
 - title-hash - sorted by hash of name
 - random - totally random
Instead of relying on the order of resources, other resources should be referenced if they are needed/related.
Resource reference/pointer:
```res_type['title']```
Example:
```Package['httpd']```
### Require
Can be used with the ```require``` attribute. The below makes sure that the Package httpd is installed
```
package {'httpd':
 ensure => installed,
}
service { 'httpd':
 ensure => running,
 require => Package['httpd']
}
```
Resources with the ```require``` metaparameter will only run if the resource they are dependant on will successfuly complete first.
### Before
```
package {'httpd':
 ensure => installed,
 before => Service['httpd'],
}
service { 'httpd':
 ensure => running,
}
```
This makes sure the package is installed before running the service httpd resource.
Require is generally more readable than before.

### Refreshable
Some resources take action based on an event occuring in a different resource, like a confiugration change - service restart.
Some resources are ```refreshable```, they take action when they receive a refresh event.
```subsribe``` and ```notify``` can be used to send an event notification.
Example:
```
file { '/etc/httpd/httpd.conf':
	ensure => file,
	source => 'puppet:///modules/apache/httpd.conf',
{

service { 'httpd':
	ensure => running,
	subsribe => File['/etc/httpd/httpd.conf'],
}
```
Subscribes behaves like ```require```. It will make sure the specified resource is managed before itself. When using ```subscribe```, when any change is reported by the specified resource a refresh is triggered. In the case of the ```service``` resource receiving a refresh event will restart the service.

```notify``` is the opposite of ```subscribe```.

```subscribe``` implies ```require``` 
```notify``` implies ```before```

Only one of the above is required.

```exec``` is also refreshable. A ```refreshonly``` parameter makes a resource run only when it receives a refresh event.

### Implied depenedncies

The ```user``` resource type autorequres any groups specified. If there is a ```group``` resource required by ```user```, it will be run before the ```user``` resource without having to use any of the relationships.
```file``` also autorequires the user specified in the owner attribute.

### Resource chaining
A shorter syntax for expressing relationships is supported. When referencing resource, they must be declared.
```
Package['httpd'] -> File ['/etc/httpd/httpd.conf']
```
In this case the package should be run before the file resource.

Resource declaration can also be chained, this is generally avoided.
```
package { 'aaaa':
	ensure => installed,
} ->
file {'aa/aa':
	ensure => file.
}
```
Syntax for chaining resources:
```->``` - left before right
```<-``` - right before left
```~>``` - left refreshes right
```<~``` - right refreshes left

# Variables
Variables are prefixed with ```$``` and assigned with ```=```.
They must begin with a lower case letter or underscore and can contain alphanumeric and underscores
Variables in puppet can't be modified or re-declared
They can be used as resource titles, values for parameters.
Strings shoudl be always quoted:
- single quotes for static content
- double quotes for interpolated content
When interpolating variables into a string, the variable should be in brackets.
Example:
```
$prefix='README'
$suffix='txt'
$filename="${prefix}.${suffix}"
```

## Arrays
Array items are declared inside square brackets
Arrays can be user in the resource title, this creates multiple resources
```
$users = ['bob', 'susan', 'peter']
user { $users:
	ensure => present,
}
```
Some resource types take arrays for their attribute. For example the groups parameter of the user resource.
References are actually arrays.
```
require => Package['httpd', 'ansible']
```
The ```require``` parameter also can take an array as the argument
```
require => [ Package['httpd'], Service['httpd']]
```
## Hashes
Hashes are declared using brackets ```{ }```
Keys and values are separated by a hashrocket ```=>```
```
$uids={
'bob' => '9999',
'susan => '1231',
'peter' => '312',
}
```
To reference a value use square brackets
```
$uid_susan = $uids['susan']
```
## Scopes
Variables exist within a scope
If a variable is not found in the current scope, the next scope will be searched.
Scoped are namespacde with ```::```, to reference the top scope varialbe use ```$::var```. If a variable would be in another scope use ```$::name_of_scope::var```.

## Facts
Facts are available in a hash called ```$::facts```, they are also accessible in the top level scope.
``` $::facts['os']['family']```
To get a list of facts use the ```facter``` command. Use ```.``` after facter to access variables within.
Puppet doesn't validate if the facts are genuine. That's why ```$::trusted``` exists, they could not have been altered and come from the node's certificate.

# Conditionals
There are 2 types of conditionals:
 - assignment conditionals: depending on the evaluation, a value of a variable will be set
 - flow conditionals: control which code gets executed during certain conditions.
Both case statements and selectors can use regular expressions.
Lower and upper case doesn't matter.
## Selectors
They assign data based on evaluation and don't control the flow of code
Example:
```
$package_name = $::facts['os']['family'] ? {
	'Debian' => 'apache2',
	'Redhat' => 'httpd',
	'Solaris' => 'CSWApache2',
	default => 'httpd',
}
```
Based on the value of the variable, the value of $package_name will be set. If nothing matches, default is used.

## Case statements
Case statemets execute a block of code depending on the evaluation
Example:
```
case $::facts['os']['family'] {
	'Redhat': {
		include yum
	}
	'Debian': {
		include apt
	}
	default: {
		fail ('Unknown operating system')
	}
}
```
Based on the value of the variable, different code blocks will be executed.

## if/else/elsif
They are used to control the general flow of code. The block is executed if the evaluation is true
Example:
```
if $install_package{
	package { $packagename:
		ensure => installed,
	}
}
```
if statements support comparing values with ```==``` and other standard operators. ````=~``` matches regular expression

```unless``` is the same as ```!=```. 

# Data types
Data types are expressed as a plain word starting wiht an uppercase letter. Data types offer a way to assert that a variable conforms to a specific type.
Depending on the data type, optional parameters can be supplied to narrow the validation criteria. For example, the String data type takes parameters of minimum and maximim lenght ```String[5,10]```.
```Float``` and ```Integer``` types support parameters for minimum nad maximum values.
```Array``` supports 3 optional parameters, the content type and the min and max length of the array. ```Any``` is acceptable for content type. ```Array[Any, 5, 15]```.
The ```Hash``` type supports four parameters. They are the same as the ones for the Array type, but also specifies the content type for the keys. ```Hash[String,Integer,5,10]```
The ```Regexp``` type ensures that the value given is a regular expression.
The ```Undef``` type asserts the value is undefined or undef.
The ```Variant``` data type asserts that a value can be one of a mixture of other types, it takes a list of data types as parameters. ```Variant[String, Integer]```
The ```Enum``` type takes a list of strings. The value of Enum always matches one of its parameters. ```Enum['yes','no']
```Optional``` takes a data type as a parameter and takes on a value that matches the data type of the parameter, or is undef.

The ```=~``` operator is used to validate a value against a data type. The outcome is true or false. ```$hostname =~ String[4]```.

Data types can be used in the evauluation of case and selector conditional statements.
```
$bar = %foo ?{
	String => 'It's a string',
	Integer => 'It's a number',
	default => 'other data type',
}
```

# Functions

Functions are server side methods and can be written in either Ruby or Puppet DSL. They are executed server side during catalog compilation. Functions either return data to assign to a variable or perform an action with no return value (statement function).

There are 2 ways of calling a function:
- Prefixed syntax: ```function(arg, arg)```
- Chained syntax: ```arg.function(arg,arg)```


Example of a function call ```notice("Hello World")``` or ```"Hello World".notice. They have the same effect, although the 1st syntax is more readable.

Functions can be found at three levels:
- Global functions - ```function```
- Environment level functions - ```environment::function```
- Module functions - ```modeulename::function```

There are 2 types of functions:
- assignment functions - returns a value to assign to a variable
- lambdas (code blocks) - they provide anonymous functions. Usually a function will yield an object(paramete)_ within the code block. Prefixed and chainged syntaxes are supported. Prefixed: ```function(args) |parameters| {}``` Chained: ```argument.function(arguments) |parameters| {}```
Example:
```
$user = with('susan', 'susan@example.com') |$u, $e| {
 {
	"user_name"=>$u,
	"email" => $e,
 }
}
```
In this example, $user will contain the hash
```
"user_name" => "susan",
"email" => "susan@example.com",
```
One of the common use of code blocks is to iterate over hashes or arrays.
Example:
```
$hosts = ['host1','host2']
$hosts.each | $v | {
file { [ "/var/sites/${v}", "/var/log/vhosts/${v}"  ]:
	ensure => directory,
	}
}
```
Code block syntax supports data type validation
```each($hosts) |String $hostname, Integer $port| {}```

## Writing functions
Functions can be written in either Ruby or the Puppet DSL. Puppet functions are usually deployed from the functions directory of the module root: ```modulepath/modulename/functions/name.pp```
Example:
```
function mymodule:funcname(String $myname) >> String {
$greeting = "Hello ${myname}, how do you do?"
$greeting
}
```
Defining resources within functions is bad practice and shouldn't be done.

Functions in Ruby should be placed in ```modulepath/lib/puppet/functions```

# Templates
They are used for serving dynamic file contents
Puppet supports two template formats:
- EPP (Embedded Puppet), native puppet dsl templates
- ERB (Enbedded Ruby), legacy ruby templates
EPP templates are called using the built-in ```epp``` function.
Templates are served from ```modulepath/module_name/templates/template.epp```
The ```epp``` function renders a template and returns the content.
The 1st argument is the locatoin of the template as ```modulename/file```.```file``` is relative to the templates folder directly under the module root. The 2nd argument is a hash of parameters to pass to the template.

## Template syntax
Templates are static content with embedded dynamic tags surrounded by ```<% .... %>```
There are 3 types of tags:
- ``` <% | ...... | %> ```parameter tag
- ```<% .... %> ```functional tag
- ```<%= ..... %>``` Expression substitution tag
Example template:
```
<% | String $role,  String $server_name | %>
Welcome to <%= $server_name %>
This machine is a <%= $role %> server
```
