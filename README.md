# Horton Assessment 
TASK 1: Data Pipelines in Hive

	a.	Install the MySQL Employees sample database (available from Employees DB on Launchpad) within Hive’s MySQL instance
	b.	Perform a full import of the ‘employees’ and ‘salaries’ tables into HDP
	c.	Create Hive tables out of them using a suitable Storage Format
	d.	Perform some cleansing using either Pig, Hive or other tool set
	e.	The ‘to_date’ column overlaps with the ‘from_date’ column in the ‘salaries’ table.  This results in the employee having two salary records on the day of the ‘to_date’ column. Fix it by decrementing the ‘to_date’ column by one day (e.g. from ‘1987-06-18’ to ‘1987-06-17’) to make each salary record exclusive.
	f.	The first salary record for an employee should reflect the day they joined the company.  However, the ‘employees.hire_date’ column doesn’t always reflect this.  Clean the data by replacing the ‘employees.hire_date’ column with the first salary record for an employee.
	g.	Determine which employee lasted less than two weeks in the job in May 1985?

Login to Ambari and Postgres

	ec2-107-20-105-52.compute-1.amazonaws.com:8080  
	postgres login: ambari/bigdata
	psql ambari -U ambari -W -p 5432
	bigdata
	\dt
	select * from metainfo;
	\q


Requires: libtirpc-devel
The command to enable the optional repo is:
yum-config-manager --enable rhui-REGION-rhel-server-optional

Update Root password and grant access

	mysql -u root
	UPDATE mysql.user SET Password=PASSWORD('changeit') WHERE User='root';
	grant all on *.* to 'root'@'ec2-107-20-105-52.compute-1.amazonaws.com';

Execute SQOOP and then load data into HDFS using SQOOP IMPORT:

	sqoop eval --connect jdbc:mysql://localhost/employees --username root  --query "select * from employees limit 2"  
	UPDATE mysql.user SET authentication_string = PASSWORD('changeit'), password_expired = 'N' WHERE User = 'root' AND Host = 'localhost';
	sqoop --options-file sqoopoptions.txt --table employees --split-by EMP_NO  --target-dir /user/root/sqoop-import/employees -m 1
	sqoop --options-file sqoopoptions.txt --table salaries --split-by EMP_NO  --target-dir /user/root/sqoop-import/salaries -m 1
		OR
	sqoop import --connect jdbc:mysql://ec2-107-20-105-52.compute-1.amazonaws.com:3306/employees --username root --password changeit --	table salaries --split-by EMP_NO  --target-dir /user/root/sqoop-import/salaries -m 1

employees=300024
salaries=2844047


TASK 2 (e):

	hive> select * from salaries where emp_no=243212;
	OK
	243212  57785   1997-07-20      1998-07-20
	243212  58134   1998-07-20      1999-07-20
	243212  58708   1999-07-20      2000-07-19
	243212  62237   2000-07-19      2001-07-19
	243212  66279   2001-07-19      2002-07-19
	243212  68998   2002-07-19      9999-01-01
	Time taken: 0.338 seconds, Fetched: 6 row(s)
	hive> select * from salaries_orc where emp_no=243212;
	OK
	243212  57785   1997-07-20      1998-07-19
	243212  58134   1998-07-20      1999-07-19
	243212  58708   1999-07-20      2000-07-18
	243212  62237   2000-07-19      2001-07-18
	243212  66279   2001-07-19      2002-07-18
	243212  68998   2002-07-19      9998-12-31

TASK 2 (f):

	select * from employees where emp_no=243212;
	OK
	243212  1954-11-06      Flemming        Highland        M       1990-02-10
	Time taken: 0.149 seconds, Fetched: 1 row(s)
	hive> select * from employees_orc where emp_no=243212;
	OK
	243212  1954-11-06      Flemming        Highland        M       1997-07-20
	Time taken: 0.101 seconds, Fetched: 1 row(s)


TASK  2 (g):

	hive> select e.first_name,e.last_name,e.emp_no from employees_orc e join (Select emp_no from salaries_orc where year(from_date)=1985 and month(from_date)=5 and datediff(to_date, from_date) < 15 ) s on e.emp_no=s.emp_no limit 5;
	Query ID = root_20170625023940_c5474f23-2e8c-487d-9619-b48e362eccdc
	Total jobs = 1
	Launching Job 1 out of 1
	Status: Running (Executing on YARN cluster with App id application_1498344035126_0038)

	--------------------------------------------------------------------------------
					VERTICES      STATUS  TOTAL  COMPLETED  RUNNING  PENDING  FAILED  KILLED
	--------------------------------------------------------------------------------
	Map 1 ..........   SUCCEEDED      6          6        0        0       0       0
	Map 2 ..........   SUCCEEDED      1          1        0        0       0       0
	--------------------------------------------------------------------------------
	VERTICES: 02/02  [==========================>>] 100%  ELAPSED TIME: 18.30 s
	--------------------------------------------------------------------------------
	OK
	Boutros McClurg 296678
	Time taken: 19.098 seconds, Fetched: 1 row(s)
	hive> select * from salaries where emp_no=296678;
	OK
	296678  69538   1985-05-11      1985-05-23
	Time taken: 0.088 seconds, Fetched: 1 row(s)


Task 3 – Streaming Architecture with Hive

	Hive
	a.	Store the twitter data in the attached file called sample_twitter_data in HDFS. The data is in json format and should not be altered. https://github.com/sakserv/hive-json/blob/master/sample_twitter_data.txt
	b.	Once the data is in HDFS, create an hcat/hive schema to be able to answer the following question: What are all the tweets by the twitter user "Aimee_Cottle"? You will need to provide the query that answers this question.

	Hint: there are multiple ways to do this, the preferred method involves org.apache.hcatalog.data.JsonSerDe - if that doesn't work search for Json serde's in the www - there are some you can compile from source to get it to work

	Streaming
	c.	Implement a storm topology that streams in tweets (https://dev.twitter.com/streaming/overview), does some interesting analytics in real-time on the tweets, and then persists into HDFS.

SOLUTION:

	Sample TWITTER DATA:
	{"user":{"userlocation":"Cinderford, Gloucestershire","id":230231618,"name":"Aimee","screenname":"Aimee_Cottle","geoenabled":true},"tweetmessage":"Gastroenteritis has pretty much killed me this week :( off work for a few days whilst I recover!","createddate":"2013-06-20T12:08:14","geolocation":null}
	{"user":{"userlocation":"Garena ID : NuraBlazee","id":635239939,"name":"Axyraf.","screenname":"Asyraf_Fauzi","geoenabled":false},"tweetmessage":"RT @abhigyantweets: Can't stop a natural disaster but so many lives in U'khand wouldn't have been lost if there was disaster preparedness. …","createddate":"2013-06-20T12:08:16","geolocation":null}
	{"user":{"userlocation":"Gemert,Netherlands","id":21418083,"name":"Ad van Steenbruggen","screenname":"torment00","geoenabled":true},"tweetmessage":"♫ Listening to 'The Constant' by 'Anthrax' from 'Worship Music","createddate":"2013-06-20T12:08:20","geolocation":null}
	{"user":{"userlocation":"","id":494156410,"name":"Balistik KartikaSari","screenname":"Balistik_Akas","geoenabled":false},"tweetmessage":"NEXT MATCH : PERSIBA BALIKPAPAN vs Persiwa Wamena | 26-06-2013 | Std.Parikesit | 15.30 Wita | NOT LIVE !!!","createddate":"2013-06-20T12:08:28","geolocation":null}
	{"user":{"userlocation":"","id":1183221613,"name":"Gabrielle Moser","screenname":"binhamoser","geoenabled":true},"tweetmessage":"Meu que merda, meu médico me proibiu de ir na passeata por que eu posso ficar com pneumonia, ah se foder","createddate":"2013-06-20T12:08:29","geolocation":null}
	{"user":{"userlocation":"Cheltenham, England","id":377932072,"name":"Sarah Daly","screenname":"sarahmygreeneye","geoenabled":false},"tweetmessage":"RT @healthpoverty: Interesting article from @foodpolicynews on the relationship between agriculture and #malaria in Africa...","createddate":"2013-06-20T12:08:39","geolocation":null}
	{"user":{"userlocation":"","id":1514008171,"name":"Auzzie Jet","screenname":"metalheadgrunge","geoenabled":false},"tweetmessage":"Anthrax - Black - Sunglasses hell yah\n http://t.co/qCNjba57Dm","createddate":"2013-06-20T12:08:44","geolocation":null}
	{"user":{"userlocation":"BANDUNG-INDONESIA","id":236827129,"name":"Powerpuff Girls","screenname":"TasyootReyoot","geoenabled":true},"tweetmessage":"RT @viddyVR: Kok gitu sih RT @TasyootReyoot: #HelloKitty itu aneh pala gede bekumis pula badan kecil kayak org kena polio\"","createddate":"2013-06-20T12:08:45","geolocation":null}
	{"user":{"userlocation":"Bolton, UK","id":14141159,"name":"Chris Beckett","screenname":"ChrisBeckett","geoenabled":true},"tweetmessage":"vCOps people - Does Advanced Edition == 5 VMs? I know Std has UI and analytics VM, but what does rest use? Hyperic etc? #vmware #vcops","createddate":"2013-06-20T12:08:46","geolocation":null}


	CREATE EXTERNAL TABLE json_serde_table (
		id string,
		person struct<email:string, first_name:string, last_name:string, location:struct<address:string, city:string, state:string, zipcode:string>, text:string, url:string>)
	ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
	LOCATION '/tmp/json/';
	
	SELECT id, person.first_name, person.last_name, person.email, 
	person.location.address, person.location.city, person.location.state, 
	person.location.zipcode, person.text, person.url
	FROM json_serde_table LIMIT 5;

	 person struct<email:string, first_name:string, last_name:string, location:struct<address:string, city:string, state:string, zipcode:string>, text:string, url:string>)

	CREATE EXTERNAL TABLE twitter_json_table(json string)
	LOCATION '/user/root/twitterjson';

	select v1.tweetmsg,v1.cdate,v1.geoloc,v2.userloc,v2.id,v2.name,v2.screenname,v2.geoenabled from twitter_json_table t LATERAL VIEW json_tuple(t.json, 'tweetmessage', 'createddate', 'geolocation', 'user') v1 as tweetmsg, cdate, geoloc, guser LATERAL VIEW json_tuple(v1.guser,'userlocation','id','name','screenname','geoenabled') v2 as userloc, id, name, screenname, geoenabled; 

	select v1.tweetmsg,v2.name,v2.screenname from twitter_json_table t LATERAL VIEW json_tuple(t.json, 'tweetmessage', 'createddate', 'geolocation', 'user') v1 as tweetmsg, cdate, geoloc, guser LATERAL VIEW json_tuple(v1.guser,'userlocation','id','name','screenname','geoenabled') v2 as userloc, id, name, screenname, geoenabled where v2.screenname='Aimee_Cottle';



	SOLUTION: TWITTER STORM:

	/usr/jdk64/jdk1.8.0_112/bin/javac -cp /usr/hdp/current/storm-client/lib/*:/root/twitterstream/lib/lib/*:/usr/hdp/2.6.1.0-129/storm/contrib/storm-hdfs/storm-hdfs-1.1.0.2.6.1.0-129.jar:/usr/hdp/2.6.1.0-129/storm/contrib/storm-autocreds/*: *.java

	nohup java -cp /usr/hdp/current/storm-client/lib/*:/root/twitterstream/lib/lib/*:/usr/hdp/2.6.1.0-129/storm/contrib/storm-hdfs/storm-hdfs-1.1.0.2.6.1.0-129.jar:/usr/hdp/2.6.1.0-129/storm/contrib/storm-autocreds/*: TwitterHashtagStorm DZXwtHv0mzD8eI4RJ9XW6Crfh MpxlGGkYcSSW8m4ZANK3dGOEZmY3kFftpg7Ve15vxGnbJ6EzD1 879108896190865410-5j8bH1ITQeuTC2D9JzU7ESpxKKpeZhO jbdVf0E7uqo5HylFucSp2XoCTj55KbIJomu12svLGqtK3 foodie cook &

	use twitterstorm;
	select * from twitterstorm;
