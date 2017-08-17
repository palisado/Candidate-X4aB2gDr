# Candidate-X4aB2gDr
WordPress with Puppet Cloudformation


Creating WordPress Sample Website using Puppet CM Automated by AWS Cloudformation Template


Introduction

This AWS Cloudformation template provides an automated solution to build WordPress website that is connected to a separate MySQL database instance on the backend. All these instances run on AWS Cloud VPC infrastructure that is built as part of the end-to-end automated solution.
The reasons for the separation between Web server and database server are for security and scalability. It is a recommended security best practice to place database server on a Internet non-accessible subnet while leaving the web server(s) on the DMZ/ Internet accessible subnet. In addition, this separation makes it possible to scale out the web server front-end in the event of high HTTP traffic loads.


Assumption

Below assumptions were made in order to simplify this project:
1. The end-to-end Cloudformation solution and all its associated resources will be run on AWS us-east-1 (Northern Virginia) region. 
2. S3 bucket named "candidate-x4ab2gdr" has been created and will be used to host Puppet configuration code (module, manifest and classes), pre-populated WordPressDB MySQL dump backup files, and the necessary WordPress configuration files.
3. For simplicity, access to "candidate-x4ab2gdr" S3 bucket does not need to be secured.  
4. Building High Available Web server is not needed because of limited user access permission (no access to Elastic LoadBalance).
5. Building MySQL server cluster in multi-AvailableZone is not needed in order to simplify the project.
6. All the end-users are located in the U.S. north east region.


Approach

The solution started with building the AWS Cloudformation template that accepts user input parameters below:
1. RemoteManagementNetwork: This is the remote network subnet where systems administrator will be SSH-ing from.
2. VPC Subnet: CIDR subnet for VPC address. Default value is "10.0.0.0/16".
3. Public Subnet 1: CIDR block for Public Subnet 1. This will be used to host WordPress Web server1.
4. Public Subnet 2: CIDR block for Public Subnet 2. This will be used to host WordPress Web server2.
5. Private Subnet 1: CIDR block for Private Subnet 1. This will be used to host MySQL database server1.
6. AccessKey : AWS AccessKey credential needed to access AWS resources.
7. ContentLocation: Location of Puppet config code such as module, manifest, and classes.
8. ContentManifest: Content of Puppet manifest.
9. DatabasePassword: This same password will be used for MySQL root and WordPressDB users.
10. DatabaseUser: This is the username for WordPressDB MySQL user and also WordPress Application Admin user. 
11. InstanceType: Choose Instance Type for the Web and database servers. The only available types are t2.micro and t2.small instance type.
12. KeyName: This is the SSH key-pair used to login to the instances.
13. S3BucketName: The name of S3 bucket that is used to host Puppet code, WordPress config file and MySQL WordPressDB backup dump file.
14. SecretKey: AWS Secret Key needed to access AWS resources.

For most part, we can leave the above parameter values as the default ones except for the RemoteManagementNetwork and KeyName.

The Cloudformation template will build infrastructure such as creating VPC, creating 3 subnets in the VPC (1 private subnet for MySQL server and 2 public subnets in different AZ for Web servers), NAT gateway, S3 VPC EndPoint, Security Groups, InternetGateway, RouteTable, etc..

Once the above infrastructure has been created, Cloudformation will start provisioning PuppetMaster instance, install all the necessary RPM packages to run Puppet Master server, setup the manifest, module, etc.. and finally start the Puppet Master service/ daemon.

We will be using Puppet plugin for AWS Cloudformation "facter cfn-init" for the puppet agent Instances.

		# cfn.rb
		require 'rubygems'
		require 'json'
		filename = "/var/lib/cfn-init/data/metadata.json"
		if File.exist?(filename)
		   parsed = JSON.load(File.new(filename))
		   parsed.default = Hash.new
		   parsed["Puppet"].each do |key, value|
			  actual_value = value
			  if value.is_a? Array
			   actual_value = value.join(',')
			  end
			  Facter.add("cfn_" + key) do
				setcode do
				  actual_value
				end
			  end
		   end
		end

The above facter cfn plugin will be used by Puppet to pull information and parameters from AWS Cloudtemplate such as Instance roles, databaseuser, databasepassword, etc... (see below "Puppet" section from CloudFormation template):

		"Puppet" : {
			 "roles" : [ "wpdb" ],
			 "database" : "WordPressDB",
			 "user" : {"Ref" : "DatabaseUser"},
			 "password" : {"Ref" : "DatabasePassword" }
		}
		
The above information, "roles" in particular, is used by puppet to determine which content manifest it needs to pull. In this case, we only have 2 roles; wordpress and wpdb. Roles "wordpress" is used to configure WordPress instance while "wpdb" is used for MySQL instance.
		
		"/etc/puppet/manifests/nodes.pp" : {
		  "content" : {"Fn::Join" : ["", [
			"node basenode {\n",
			" include cfn\n",
			"}\n",
			"node /^.*internal$/ inherits basenode {\n",
			" case $cfn_roles {\n",
			" ", { "Ref" : "ContentManifest" }, "\n",
			" /wpdb/: { include wpdb } \n",
			" }\n",
			"}\n"]]},
								
Once PuppetMaster instance is up and operational, Cloudformation will instantiate DatabaseInstance and Webserver instances, install necessary RPM packages and configuration files, and finally bring up puppet agent service/ daemon on those instances.

On the DatabaseInstance, Puppet will install the latest "mysql-server" rpm package and make sure it is "running", will pull down WordPressDB.dmp MySQL backup dump file, and then will run once for restoring WordPressDB.dmp and setting up mysql root admin password. The Puppet "run once" is achived by using "creates =>" attributes within "exec" resource type. "creates =>" attribute will check if the file doesn't exist it will execute the command. In this case, we will create a hidden file /tmp/.WordPressDB_dmp.done at the end of the "command" attribute so that the same command will not be executed again.

		class wpdb {
		  package { mysql-server:
				ensure => latest
			}
		  file { "/tmp/WordPressDB.dmp":
				content => template('wpdb/WordPressDB.dmp'),
				require => Package["mysql-server"]
		  }
		  file { "/tmp/wpdb-config.sql":
				content => template('wpdb/wpdb-config.erb'),
				require => Package["mysql-server"]
		  }
		  
		  service { mysqld:
				enable => true,
				ensure => "running",
				require => Package["mysql-server"]
		  }
		  
		  exec { "run once restore WordPressDB":
				command => "mysqladmin -u root password $cfn_password; mysql -uroot -p$cfn_password < /tmp/WordPressDB.dmp; mysql -uroot -p$cfn_password < /tmp/wpdb-config.sql; touch /tmp/.WordPressDB_dmp.done",
				require => [ File["/tmp/WordPressDB.dmp"], File["/tmp/wpdb-config.sql"] ],
				path => ['/usr/bin', '/bin', '/sbin'],
				creates => '/tmp/.WordPressDB_dmp.done'
		  }
		}

On the WebserverInstance, by using the roles="/wordpress/", Puppet will pull the wordpress manifest and classes to install all the necessary rpm packages for wordpress, pull down the wordpress configuration files, and finally start the httpd daemon. 

		class wordpress {
		  package { php: ensure => latest }
		  package { php-mysql:
				ensure => latest,
				require => Package["php"] }
		  package { wordpress:
				ensure => latest,
				require => Package["php-mysql"] }
		  file { "/etc/wordpress/wp-config.php":
				content => template('wordpress/wp-config.erb'),
				require => Package["wordpress"]
		  }
		  file { "/etc/httpd/conf.d/wordpress.conf":
				content => template('wordpress/wordpress.erb'),
				require => Package["wordpress"]
		  }
		  file { "/tmp/wp-mysql_runonce.sql":
				content => template('wordpress/wp-mysql_runonce.erb'),
				require => Package["wordpress"]
		  }

		  exec { "run once fix WordPressDB siteurl":
				command => "mysql -u$cfn_user -p$cfn_password -h$cfn_host < /tmp/wp-mysql_runonce.sql; touch /tmp/.WordPressMysql.done",
				require => File["/tmp/wp-mysql_runonce.sql"],
				path => ['/usr/bin', '/bin', '/sbin'],
				creates => '/tmp/.WordPressMysql.done'
		  }
		  
		  service { httpd:
				enable => true,
				ensure => "running",
				require => File["/etc/wordpress/wp-config.php"]
		  }
		}

Once all the instances are up and operational, Cloudformation template will output the WordPress webserver URL in the "output" tab.
NOTE: Please wait for few minutes before accessing the WordPress site URL in the output tab after Cloudformation has completed its job. We need to make sure that Puppet has completed all its necessary tasks to setup and bring up the Database and Webservers.


	
Suggestion

In order to create Highly Available (HA) and scalable WordPress site, the assigned IAM user needs some additional permissions such as EFS, Elastic LoadBalance, and AutoScaling. It will also be beneficial if the IAM user also has permission to Amazon RDS MySQL database, Route 53, and CDN.
Amazon RDS provides redundant and scalable MySQL database server while AutoScaling and Elastic LoadBalance provide local HA for the WordPress webservers. For the WordPress webserver content, we may want to use Amazon EFS to create wordpress NFS mount partition on the webserver instances. This way if the users uploaded new site contents/ files, those files will be available immediately to the rest of the webservers in the cluster via NFS mount.

For the global geographical HA, we could incorporate Route 53 routing policy such as failover, latency, weighted, or geolocation routing policy to route the end-user traffics to the closest AWS region.
