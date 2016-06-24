![logo](http://mysqlhighavailability.com/wp-content/uploads/2015/12/MySQL_Cluster_400.png)

# What is MySQL Cluster?

MySQL Cluster is a technology that enables clustering of in-memory databases in a shared-nothing system. The shared-nothing architecture enables the system to work with very inexpensive hardware, and with a minimum of specific requirements for hardware or software.

MySQL Cluster is designed not to have any single point of failure. In a shared-nothing system, each component is expected to have its own memory and disk, and the use of shared storage mechanisms such as network shares, network file systems, and SANs is not recommended or supported.

MySQL Cluster integrates the standard MySQL server with an in-memory clustered storage engine called NDB (which stands for “Network DataBase”). In our documentation, the term NDB refers to the part of the setup that is specific to the storage engine, whereas “MySQL Cluster” refers to the combination of one or more MySQL servers with the NDB storage engine.

For more information and related downloads for MySQL Cluster and other MySQL products, please visit http://www.mysql.com.

# MySQL Cluster Docker Image

This is an experimental MySQL Cluster Docker image, created and maintained by Matt Lord. The available version is:

    MySQL Cluster 7.5, the latest version (tag: 7.5 or latest)

# How to Use the MySQL Cluster Image

## Bootstrap a new Cluster by creating a management node 

    docker run -d --name=ndb_mgmd --net=host -e NODE_TYPE=management -e BOOTSTRAP=1 mattalord/mysql-cluster:7.5 

## Add a MySQL Cluster Data node to the Cluster 

Start and add a MySQL Cluster Data Node as follows:

    docker run -d --name datanode1 -e NODE_TYPE=data -e MANAGEMENT_SERVER=ndb_mgmd --link ndb_mgmd mattalord/mysql-cluster:7.5

## Add a MySQL Server node to the Cluster 

Start a MySQL instance as follows (but make sure you also read the sections *Secure Container Startup* and *Where to Store Data* below):

    docker run -d --name mysqlnode1 -e NODE_TYPE=sql -e MANAGEMENT_SERVER=ndb_mgmd -e MYSQL_ROOT_PASSWORD=my-secret-pw --link ndb_mgmd mattalord/mysql-cluster:7.5

## Connect to MySQL from an Application in Another Docker Container

This image exposes the standard MySQL port (3306), so container linking makes the MySQL instance available to other application containers. Start your application container like this in order to link it to the MySQL container:

    docker run -d --name app-container-name --link my-container-name:mysql app-that-uses-mysql

## Connect to MySQL from the MySQL Command Line Client

The following command starts another MySQL Server container node and runs the `mysql` command line client against your original MySQL container, allowing you to execute SQL statements against your database:

    docker run -it --link my-container-name:mysql -e MANAGEMENT_SERVER=ndb_mgmd --link ndb_mgmd -e MYSQL_ROOT_PASSWORD=my-secret-pw --rm mattalord/mysql-cluster:7.5 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'

... where `my-container-name` is the name of your original MySQL Cluster container.

More information about the MySQL command line client can be found in the MySQL reference documentation at http://dev.mysql.com/doc/refman/en/

## Container Shell Access and Viewing MySQL Log Files

The `docker exec` command allows you to run commands inside a Docker container. The following command line will give you a bash shell inside your MySQL container:

    docker exec -it my-container-name bash

The MySQL Server log is located at `/var/log/mysqld.log` inside the container, and the following command line from a shell inside the container will let you inspect it:

    more /var/log/mysqld.log

The MySQL Cluster Management node logs and other data are located at `/var/lib/ndb/management`.

The MySQL Cluster Data node logs and other data are located at `/var/lib/ndb/data`.

# Environment Variables

When you start the MySQL Cluster image, you can adjust the configuration of the MySQL instance by passing one or more environment variables on the `docker run` command line. Do note that none of the variables below will have any effect if you start the container with a data directory that already contains a database: any pre-existing database will always be left untouched on container startup.

## `NODE_TYPE`

This variable specifies the type of node that this process will be in the Cluster. It has to be one of: 'management', 'sql', or 'data'.

## `BOOTSTRAP`

This variable is only used with NODE_TYPE=management to clarify that this is the first/only management node and is bootstraping a new Cluster.

## `MANAGEMENT_SERVER`

This variable specifies the hostname or IP address of the Cluster Management server instance for this Cluster.

## `MYSQL_ROOT_PASSWORD`

This variable specifies a password that will be set for the MySQL root superuser account. In the above example, it was set `to my-secret-pw`. **NOTE:** Setting the MySQL root user password on the command line is insecure. See the section *Secure Container Startup* below for an alternative.

As an alternative to specifying the password explicitly, if the variable is set to a file path, the contents of the file will be used as the root password.

## `MYSQL_RANDOM_ROOT_PASSWORD`

When this variable is set to `yes`, a random password for the server's root user will be generated. The password will be printed to stdout in the container, and it can be obtained by using the command `docker logs my-container-name`.

## `MYSQL_ONETIME_PASSWORD`

This variable is optional. When set to `yes`, the root user's password will be set as expired, and must be changed before MySQL can be used normally. This is only supported by MySQL 5.6 or newer.

## `MYSQL_DATABASE`

This variable is optional. It allows you to specify the name of a database to be created on image startup. If a user/password was supplied (see below) then that user will be granted superuser access (corresponding to GRANT ALL) to this database.

## `MYSQL_USER`, `MYSQL_PASSWORD`

These variables are optional, used in conjunction to create a new user and set that user's password. This user will be granted superuser permissions (see above) for the database specified by the `MYSQL_DATABASE` variable. Both variables are required for a user to be created.
 
Do note that there is no need to use this mechanism to create the `root` superuser, that user gets created by default with the password set by either of the mechanisms (given or generated) discussed above.

## `MYSQL_ALLOW_EMPTY_PASSWORD`

Set to `yes` to allow the container to be started with a blank password for the root user. **NOTE:** Setting this variable to `yes` is not recommended unless you really know what you are doing, since this will leave your MySQL instance completely unprotected, allowing anyone to gain complete superuser access.

# Notes, Tips, Gotchas

## Where to Store Data

There are basically two ways to store data used by applications that run in Docker containers. We encourage users of MySQL with Docker to familiarize themselves with the options available, including:

* Let Docker manage the storage of your database data by writing the database files to disk on the host system using its own internal volume management. This is the default and is easy and fairly transparent to the user. The downside is that the files may be hard to locate for tools and applications that run directly on the host system, i.e. outside containers.
* Create a data directory on the host system (outside the container) and mount this to a directory visible from inside the container. This places the database files in a known location on the host system, and makes it easy for tools and applications on the host system to access the files. The downside is that the user needs to make sure that the directory exists, and that e.g. directory permissions and other security mechanisms on the host system are set up correctly.

The Docker documentation is a good starting point for understanding the different storage options and variations, and there are multiple blog and forum postings that discuss and give advice in this area. We will simply show the basic procedure here for the latter option above:

1. Create a data directory on a suitable volume on your host system, e.g. `/my/own/datadir`.
2. Start your MySQL container like this:

```
    docker run --name my-container-name -v /my/own/datadir:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mattalord/mysql-cluster:7.5
```

The `-v /my/own/datadir:/var/lib/mysql` part of the command mounts the `/my/own/datadir` directory from the underlying host system as `/var/lib/mysql` inside the container, where MySQL by default will write its data files.

Note that users on systems with SELinux enabled may experience problems with this. The current workaround is to assign the relevant SELinux policy type to the new data directory so that the container will be allowed to access it:

    chcon -Rt svirt_sandbox_file_t /my/own/datadir

## Usage Against an Existing Database

If you start your MySQL container instance with a data directory that already contains a database (specifically, a `mysql` subdirectory), the `$MYSQL_ROOT_PASSWORD` variable should be omitted from the `docker run` command line; it will in any case be ignored, and the pre-existing database will not be changed in any way.

## Port forwarding

Docker allows mapping of ports on the container to ports on the host system by using the -p option. If you start the container as follows, you can connect to the database by connecting your client to a port on the host machine, in this example port 6603:

    docker run --name my-container-name -p 6603:3306 -d mattalord/mysql-cluster:7.5

## Passing options to the node process

You can pass arbitrary command line options to the MySQL server by appending them to the `run command`:

    docker run --name my-container-name -d mattalord/mysql-cluster:7.5 --option1=value --option2=value

In this case, the values of option1 and option2 will be passed directly to the server when it is started. The following command will for instance start your container with UTF-8 as the default setting for character set and collation for all databases in MySQL:

    docker run --name my-container-name -d mysql/mysql-server --character-set-server=utf8 --collation-server=utf8_general_ci

## Using a Custom MySQL Config File

The MySQL startup configuration in these Docker images is specified in the file `/etc/my.cnf`. If you want to customize this configuration for your own purposes, you can create your alternative configuration file in a directory on the host machine and then mount this file in the appropriate location inside the MySQL container, effectively replacing the standard configuration file.

If you want to base your changes on the standard configuration file, start your MySQL container in the standard way described above, then do:

    docker exec -it my-container-name cat /etc/my.cnf > /my/custom/config-file

... where ´/my/custom/config-file´ is the path and name of the new configuration file. Then start a new MySQL container like this:

    docker run --name my-new-container-name -v /my/custom/config-file:/etc/my.cnf -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mattalord/mysql-cluster:7.5

This will start a new MySQL container ´my-new-container-name´ where the MySQL instance uses the startup options specified in ´/my/custom/config-file´.

Note that users on systems where SELinux is enabled may experience problems with this. The current workaround is to assign the relevant SELinux policy type to your new config file so that the container will be allowed to mount it:

    chcon -Rt svirt_sandbox_file_t /my/custom/config-file
