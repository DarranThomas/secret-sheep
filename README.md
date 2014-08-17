MOD_MYSQL_ACCESSLOG.C for LightTPD up to version 1.4.35
============

This is the MOD_MYSQL_ACCESSLOG code that allows LIGHTTPD to write its logs to a DATABASE instead of disk files.



Here is how you get it to work.....




Prerequisites:
========================================================
1.SOURCE distribution of lighttpd. Here we used version, "1.4.35".
http://download.lighttpd.net/lighttpd/releases-1.4.x/lighttpd-1.4.35.tar.gz

2.And you need mariadb headers and mariadb database / or MYSQL etc.


Procedure:
========================================================
*** STEP 0 *** : Please decompress the lighttpd source and get into the directory.
tar -xzf lighttpd-1.4.35.tar.gz
cd lighttpd-1.4.35


*** STEP 1 *** : 
sudo vi configure.ac
look for the plugins="mod_mysql_vhost" section (somewhere around line 668). 
Now put the following code in a new para at around line 681, like the others.
# DJT - Added the facility to directly write the access log out to the database.  This allows the tracking of pixel downloads
plugins="mod_mysql_accesslog"
if test ! "x$MYSQL_LIBS" = x; then
  do_build="$do_build $plugins"
else
  no_build="$no_build $plugins"
fi


*** STEP 2 ***
sudo vi src/Makefile.am.
Add the following lines somewhere near the other module definitions (like, lib_LTLIBRARIES += mod_mysql_vhost.la) - AROUND LINE 138
# DJT - Added the facility to directly write the access log out to the database.  This allows the tracking of pixel downloads
lib_LTLIBRARIES += mod_mysql_accesslog.la
mod_mysql_accesslog_la_SOURCES = mod_mysql_accesslog.c
mod_mysql_accesslog_la_LDFLAGS = -module -export-dynamic -avoid-version -no-undefined
mod_mysql_accesslog_la_LIBADD = $(MYSQL_LIBS) $(common_libadd)
mod_mysql_accesslog_la_CPPFLAGS = $(MYSQL_INCLUDE)


================= STEP 3 FOR LIGHTTPD 1.4.X ================= 
*** STEP 3 ***. I am not sure it is needed, but you can edit the src/SConscript file at around LINE 79 INSIDE THE END BRACKET }
'mod_mysql_vhost' : { 'src' : [ 'mod_mysql_vhost.c' ], 'lib' : [ env['LIBMYSQL'] ] },
'mod_mysql_accesslog' : { 'src' : [ 'mod_mysql_accesslog.c' ], 'lib' : [ env['LIBMYSQL'] ] },


*** STEP 4 ***. Copy mod_mysql_accesslog.c into the src directory.


*** STEP 5 *** ....Make sure that libtool and gcc are installed as well as the mariaDB headers, i.e.
sudo apt-get install build-essential
sudo apt-get install libtool
sudo apt-get install automake
sudo apt-get install libmariadbclient-dev	 (MariaDB Headers)
sudo apt-get install libpcre3-dev  			 (PERL development regex functions)
sudo apt-get install  libbz2-dev			 (BZ2 Ziplib)
sudo apt-get install make
sudo apt-get install pkg-config				 should get everything required.....

sudo apt-get install libglib2.0-dev			 FOR LIGHTTPD 1.5
sudo apt-get build-dep lighttpd				 WHEN IT ALL FAILS AND NOTHING WORKS!  CARE!!


From the base directory of lighttpd (lighttpd-1.4.35) execute the following:
a)	sudo sh ./autogen.sh
b)	sudo sh ./configure --with-mysql              (--with-linux-aio)


*** STEP 6 ***. Make sure that you see the following lines ,
Plugins:
enabled: 
  mod_mysql_accesslog


*** STEP 7 ***. Now run the following commands.
sudo make clean
sudo make
***** MAKE SURE THAT THERE IS NO CURRENTLY INSTALLED VERSION OF LIGHTTPD! *****
sudo service lighttpd stop
sudo apt-get remove lighttpd
***NOW INSTALL THE NEW VERSION***
sudo make install

*** STEP 8  - USE THE CREATED DLL FILE (check first - perhaps it will run without) ***
*** The ".SO" FILE HAS BEEN CREATED AT /src/.libs/ - filename is "mod_mysql_accesslog.so"
cd /home/dataportcullis/lighttpd-1.4.35/src/.libs
sudo cp mod_mysql_accesslog.* /usr/lib/lighttpd/
sudo service lighttpd start  (then check the error.log at "/var/log/lighttpd/")

******** CHANGE lighttpd within /etc/init.d AT LINE 18 **************
#DAEMON=/usr/sbin/lighttpd
DAEMON=/usr/local/sbin/lighttpd


***STEP 9 ***. Create the database table

CREATE TABLE accesslog (
  remote_host int(10) unsigned,
  remote_user varchar(2048),
  timestamp int(10) unsigned,
  request_line varchar(2048),
  status smallint(5) unsigned,
  bytes_body int(10) unsigned,
  header varchar(2048),
  environment varchar(2048),
  filename varchar(2048),
  request_protocol enum('VERSION_1_0','VERSION_1_1'),
  request_method enum('GET','POST','HEAD','OPTIONS','PROPFIND','MKCOL','PUT','DELETE','COPY','MOVE','PROPPATCH','REPORT','CHECKOUT','CHECKIN','VERSION_CONTROL','UNCHECKOUT','MKACTIVITY','MERGE','LOCK','UNLOCK','LABEL','CONNECT'),
  server_port smallint(5) unsigned,
  query_string varchar(2048),
  time_used smallint(5) unsigned,
  url varchar(2048),
  server_name varchar(2048),
  http_host varchar(2048),
  keep_alive tinyint(3) unsigned,
  bytes_in int(10) unsigned,
  bytes_out int(10) unsigned,
  response_header varchar(2048)
)

10. Apply the following configuration to your lighttpd.conf

#########################################
# MySQL Accesslog authentication
######################################## ENTIRE SECTION READY TO BE UNCOMMENTED FOR THE PIXEL DOWNLOAD TEST
mysql-accesslog.user = "accesslog_USER"
mysql-accesslog.pass = "accesslog"

# MySQL database
mysql-accesslog.data = "LightTPD"
#mysql-accesslog.data = "pipe"  (not needed - probably)

# MySQL socket or host
mysql-accesslog.sock = "/var/run/mysqld/mysqld.sock"
mysql-accesslog.host = "localhost"

# MySQL query to be executed
mysql-accesslog.query = "INSERT INTO accesslog SET remote_host=%h, timestamp=%t, status=%s, header=%{User-Agent}i, query_string=%q, url=%U"
#########################################

