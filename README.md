
CoolBeans is an Inversion of Control (IOC) / Dependency Injection (DI) library for Node.js. CoolBeans is loosely based
on ColdSpring for ColdFusion and Spring IOC for Java.  It's a single js file and currently appears to be quick and easy.

To use CoolBeans you simply create an instance of the CoolBeans and load the configuration file like this:

	var cb = require("CoolBeans");
	cb = new cb("./config/dev.json");

The above code is the only require function you should need in your entire application.  Once you've required CoolBeans
you need to create a new instance and pass in the path to its' configuration.  This is shown above.

Once you have the fully loaded CoolBeans you can use it to quickly create fully configured singleton objects based on
its' configuration.  The config file for CoolBeans is a JSON file so the entire thing is wrapped in {}.

Each element in the root of the configuration file is a bean (bean = Java for object) that CoolBeans can create.  Here's an
example:

	{
		"fs": {"module": "fs"}
	}

This is essentially the same as:

	var fs = require("fs");

However, we now only need to define this one time for an application, rather than in each file that requires it.

You can also specify paths to modules that are not node_modules. For example:

	{
		"Recipient": {"module": "../entities/recipient"}
	}

At the most basic, the above means that CoolBeans will call require for the module and cache the results in a variable named
Recipient.

You can get any of the configured beans by calling cb.get("beanName") where beanName is the name of the bean you want
to get.  For example:

	cb.get("Recipient");

The above will lazily create the Recipient bean, cache it as a singleton, and return it.

You can get a lot more complex with configuration too.  For example, you can specify if CoolBeans should call a constructor
and what arguments to pass into the constructor.  For example:

	"codeGenerator": {
		"module": "../util/codeGenerator",
		"constructorArgs": [
			"foo",
			123
		]
	}

What the above is saying is that when we get the codeGenerator bean, we need to load the module specified then call
new on the module and pass in the values specified in the constructorArgs section to the constructor. The above means:

	new codeGenerator("foo", 123);

You can also specify more complex values to pass into constructor arguments:

	"codeGenerator": {
		"module": "../util/codeGenerator",
		"constructorArgs": [
			"foo",
			123,
			{"value": [1, 2, 3]},
			{"value":
				{
					"foo": "bar",
					"bar": "foo"
				}
			}
		]
	}

Note that while you can in a string without indicating explicitly it's a "value", for arrays and anonymous objects you
need to provide an object with a property named "value" whose value is the value you're trying to pass in.  The above
could be written more explicitly as:

	"codeGenerator": {
		"module": "../util/codeGenerator",
		"constructorArgs": [
			{"value": "foo"},
			{"value": 123},
			{"value": [1, 2, 3]},
			{"value":
				{
					"foo": "bar",
					"bar": "foo"
				}
			}
		]
	}

Strings, numbers, arrays, and anonymous objects are not the only things you can pass into constructors.  You can also
specify other beans that could be passed in.  For example, let's say we had a database configuration object you want to
pass into any object that is used to access data you could do the following:

	"mysql": {"module": "mysql"},
	
	"dbConfig": {
		"properties": {
			"host": "server.hostname.com",
			"port": 3306,
			"user": "mysqlUser",
			"password": "password123",
			"database": "foobar"
		}
	},
	
	"recipientDao": {
		"module": "../db/recipientDao",
		"constructorArgs": [
			{"bean": "dbConfig"},
			{"bean": "mysql"}
		]
	}

The mysql bean is simply the same as saying require("mysql").  The dbConfig is an anonymous object with properties
specified (more on this in a bit).  When the recipientDao (dao = data access object) CoolBeans will see the "bean" property
and will create and pass into the constructor the fully-constructed dbConfig object and the mysql object. Here's what
that recipientDao might look like:

	module.exports = function(dbConfig, mysql){
	
		this.listRecipients = function(userId, callback){
			var client = mysql.createClient(dbConfig);
			client.query(
				"SELECT id, name, addressLine1, IfNull(addressLine2, '') as addressLine2, city, state, zip, taxDeductible, created, updated, 0 as netDonations " +
				"FROM recipient " +
				"WHERE userId = ? AND deleted = 0 "+
				"ORDER BY name",
			[userId],
			function(err, results, fields){
				client.end();
				callback(results);
			});
		}
	}

Note that there are no require statements.  The object just gets its dependencies when it's instantiated and can
immediately use them.  These dependencies are also automatically singletons.

Also note that if you want to use a transient object you would still create an instance of it the way you always have.

I also mentioned above that CoolBeans can be used to create create and populate anonymous objects.  For example:

	"dbConfig": {
		"properties": {
			"host": "server.hostname.com",
			"port": 3306,
			"user": "mysqlUser",
			"password": "password123",
			"database": "foobar"
		}
	}

This is a somewhat long-winded way of saying

	dbConfig = {
		"host": "server.hostname.com",
		"port": 3306,
		"user": "mysqlUser",
		"password": "password123",
		"database": "foobar"
	};

However, once this object is configured in CoolBeans you can easily pass it into other objects when they are created.

You can also specify properties for not-anonymous objects. You can also mix and match constructorArgs and properties.
For example:

	"creditCardDao": {
		"module": "../db/creditCardDao",
		"constructorArgs": [
			{"bean": "dbConfig"},
			{"bean": "authorize"},
			{"bean": "mysql"},
			{"bean": "CreditCard"}
		],
		"properties": {
			"service": {"bean": "service"}
		}
	}

When CoolBeans creates the creditCardDao it will first load all the beans specified in the constructorArgs. It will then
create the creditCardDao and pass in the four already-created beans to the constructor.  Once the object is constructed
it will set the service property on the object to the specified service bean.  Note, CoolBeans will look for a setter and use
that if it can find it.  For example, the service property above will first look for a function named setservice (note
that this is case sensitive).  If it can find it, it will pass in the service bean to that function.  If not, it will
simply set a public property on the object.

There are a few other interesting capabilities of CoolBeans:

Beans don't have to be lazily loaded.  You can set a bean to load when the container loads.  For example:

	"dateFormat": {
		"module": "../util/dateFormat",
		"lazy": false
	}

Also, if you have a factory that is used to construct other objects, you can specify this using the factoryBean and
factoryMethod properties.  For example:

	"knox": {"module": "knox"},
	
	"s3client": {
		"factoryBean": "knox",
		"factoryMethod": "createClient",
		"constructorArgs": [
			{"value":
				{
					"key": "myKey",
					"secret": "mySecret",
					"bucket": "myBucket"
				}
			}
		]
	}

The above s3client bean is configured so CoolBeans uses knox to create it.  The constructor args are passed into the
factoryMethod as if it were a constructor.  The above essentially boils down to:

	s3client = knox.createClient({
		"key": "myKey",
		"secret": "mySecret",
		"bucket": "myBucket"
	});

The really nice thing about CoolBeans is that it lets the objects in your system stay focused on what they do best.  It
shouldn't be your object's responsibility to know what they need to work.  They should simply get what they need to work
when they're created.  CoolBeans also helps avoid situations in complex apps where you have dozens of lines of code just
getting dependencies created just to create one object that otherwise happens to have a lot of dependencies. Lastly, CoolBeans
allows you to easily change how your application is configured in different environments.
