# hadoop-setup

As root, add your system user (ex: my_user_id) to sudo without a password
```
sudo visudo
```
add:
```
my_user_id  ALL=(ALL) NOPASSWD:ALL
```
Save and exit


# AS ROOT, download and set up openjdk in /opt/jdk/current/
sudo su
cd
wget https://download.java.net/java/GA/jdk11/13/GPL/openjdk-11.0.1_linux-x64_bin.tar.gz
tar xzvf openjdk-11.0.1_linux-x64_bin.tar.gz
rm openjdk-11.0.1_linux-x64_bin.tar.gz
mkdir /opt/jdk
mv jdk-11.0.1 /opt/jdk/
cd /opt/jdk/
ln -s jdk-11.0.1/ current
ls -la
exit


# AS THE USER, add your java home:
nano ~/.bashrc
# at the bottom of the file, add
export JAVA_HOME=/opt/jdk/current
export PATH=/opt/jdk/current/bin:$PATH
# reload it
source ~/.bashrc


# download hadoop
wget http://apache.claz.org/hadoop/common/hadoop-2.8.5/hadoop-2.8.5.tar.gz
tar xzvf hadoop-2.8.5.tar.gz
rm hadoop-2.8.5.tar.gz
sudo mv hadoop-2.8.5/ /opt/
cd /opt/hadoop-2.8.5/


# Copy example files
cd /opt/hadoop-2.8.5/
mkdir input
cp etc/hadoop/* input/

# test that Hadoop works
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.8.5.jar grep input output 'dfs[a-z.]+'
ls -l output/


# AS THE USER, CREATE A KEY so that you can ssh in locally without a password
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
<ENTER>
<ENTER>
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# TEST that you can connect without a password
ssh localhost
yes
exit


# Edit the hadoop environment to set up the java home
nano /opt/hadoop-2.8.5/etc/hadoop/hadoop-env.sh
# and change 
export JAVA_HOME=${JAVA_HOME}
# to
export JAVA_HOME=/opt/jdk/current


# Edit the hadoop core site and add the file system
nano /opt/hadoop-2.8.5/etc/hadoop/core-site.xml
# change
<configuration>
</configuration>
# to
<configuration>
 <property>
  <name>fs.defaultFS</name>
  <value>hdfs://localhost:9000</value>
 </property>
</configuration>


# Edit the hdfs config
nano /opt/hadoop-2.8.5/etc/hadoop/hdfs-site.xml
# change to replicate:
<configuration>
        <property>
                <name>dfs.replication</name>
                <value>1</value>
        </property>
</configuration>


# Edit map reduce framework config, file may not exist, so touch it first.
touch /opt/hadoop-2.8.5/etc/hadoop/mapred-site.xml
nano /opt/hadoop-2.8.5/etc/hadoop/mapred-site.xml
# contents are yarn, for resource negotiation, instead of local or classic.
<configuration>
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
</configuration>


# Edit the yarn config for the mapreduce framework:
nano /opt/hadoop-2.8.5/etc/hadoop/yarn-site.xml
# contents:
<configuration>
        <!-- Site specific YARN configuration properties -->
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
</configuration>


# Format the name node, 
cd /opt/hadoop-2.8.5/
bin/hdfs namenode -format
# Start the master and slave node of the distributed file system
sbin/start-dfs.sh
# when prompted to continue reconnecting, say yes so that it adds the key of host 0.0.0.0
yes
# ignore all the warnings


# Check the installation:
# If you're in VirtualBox, add a mapping of port 50070 to 50070, then hit http://localhost:50070
http://localhost:50070


# Create a home dir in HDFS with your linux username in the bin/hdfs/user directory
bin/hdfs dfs -mkdir /user
bin/hdfs dfs -mkdir /user/${USER}
# check that it was created
bin/hdfs dfs -ls /user/

# confirm that jps is running:
jps

# start yarn, the resource negotiator
sbin/start-yarn.sh

# run jps again, to see 2 new processes running, the NodeManager and the ResourceManager
jps


# if you're in VirtualBox, add a port of 8088 and 8088 and hit http://localhost:8088 to see the cluster view
http://localhost:8088


# create an input directory
bin/hdfs dfs -mkdir input
# copy hadoop files in there
bin/hdfs dfs -put etc/hadoop/* input/
# list that they're there
bin/hdfs dfs -ls input/


# run mapreduce
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.8.5.jar grep input output 'dfs[a-z.]+'
# watch the progress of the job here:
http://localhost:8088/cluster/


# when it's done, check the progress:
bin/hdfs dfs -cat output/*
