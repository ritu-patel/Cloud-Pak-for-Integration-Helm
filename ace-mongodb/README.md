##  How to connect to mongodb using loopback connector from ACE  

Use case: 
1. Save JSON data from a rest api to event streams
2. Get JSON data from event streams and save them to mongodb

In this use case, mongodb used is off-prem but you can spin up a mongo container/pod and grab those credentials and supply that to the datasources.json file. A custom ace docker image is built so we can have access to the loopback connectors (that the base ace image doesn't have access to) and the newly created local datasources json file.

Steps
1.  [Create a topic in Event Streams and download the certificate](#1-eventstreams)
2.  [Create an ace flow](#2-ACE-flow-in-the-toolkit)
- 2.1 [Flow to save data to EventStreams using rest api](#21-Save-data-to-EventStreams)
- 2.2 [Flow to save data from EventStreams to MongoDB](#22-Save-data-from-EventStreams-to-MongoDB)
3.  [Create a custom image of ace to deploy on openshift](#3-ACE-Custom-Image)
- 3.1 [Datasource configuration without SSL](#31-Datasource-configuration-without-SSL)
- 3.2 [Datasource configuration with SSL](#32-Datasource-configuration-with-SSL)
- 3.3 [Build and Deploy Image](#33-Build-and-Deploy-Image)
4.  [Deploy the bar file on CP4I and generate a secret](#4-Deploy-the-bar-file-on-CP4I)
5.  [Test](#5-Test)


### 1. EventStreams

1\. Click on "Create a topic" and name it \"ace\". Select 3 partitions
and leave rest the default.

![A screenshot of a social media post Description automatically
generated](./myMediaFolder/media/image1.png)

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image2.png)

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image3.png)

![A screenshot of a social media post Description automatically
generated](./myMediaFolder/media/image4.png)

2\. Click on the "connect to this topic" -\> Generate API Key

Create an api with the name \"ace\_api\" and DOWNLOAD JSON. 

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image5.png)

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image6.png)

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image7.png)

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image8.png)

3\. Download Java truststore file

![A screenshot of a social media post Description automatically
generated](./myMediaFolder/media/image9.png)



### 2. ACE flow in the toolkit

#### 2.1 Save data to EventStreams

1\. Create a Rest API project by clicking File -\> New -\> REST API and
Name the project as \"Example\"

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image10.png)

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image11.png)

2\. Click on \"+\" under Resources and select POST and name the path as
\"producer\"

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image12.png)

![A screenshot of a social media post Description automatically
generated](./myMediaFolder/media/image13.png)

3\. Save

4\. Click on the button below to add a subflow
![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image13-2.png)

4\. Add subflow and the following nodes

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image14.png)

"Out" from kafkaProducer node should go to output

![A picture containing screenshot Description automatically
generated](./myMediaFolder/media/image15.png)

"Failure" from kafkaProducer node should go to Trace Exceptions!

![A
screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image16.png)

Click on KafkaProducer -\> Under Properties -\> Select Basic and fill in
your information.

For example:

Topic Name: topic\_name

Bootstrap servers:
es-1-ibm-es-proxy-route-bootstrap-eventstreams.apps.ocp43.vg.gse-ocp.net:443

![A screenshot of a social media post Description automatically
generated](./myMediaFolder/media/image17.png)

Under Security

Security Protocol: Select SASL\_SSL

SSL protocol: TLSv1.2

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image18.png)

4\. Configure Trace Exceptions as below

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image19.png)

5\. Save


#### 2.2 Save data from EventStreams to MongoDB

1\. Right Click Flows -\> New -\> Message Flow

Name as consumer under schema gen

![A screenshot of a computer Description automatically
generated](./myMediaFolder/media/image20.png)

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image21.png)

2\. Add the following node with the kafka Properties

![A screenshot of a social media post Description automatically
generated](./myMediaFolder/media/image22.png)

![A screenshot of a social media post Description automatically
generated](./myMediaFolder/media/image23.png)

Under Security:

Security Protocol: Select SASL\_SSL

SSL protocol: TLSv1.2

3\. Configure Trace Execeptions like below:

![A screenshot of a social media post Description automatically
generated](./myMediaFolder/media/image24.png)

4\. Lets change the LoopbackRequest name as Mongo and add the following
configuration under Basic

- Data Source name: mongodb
- LoopBack Object: items

LoopBack Object("items") is the collection name in the mongodb and data source name(mongodb) is a key that is referred in the datasources.json

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image25.png)

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image26.png)

SAVE THE FLOW AS BAR FILE

File -\> NEW -\> BAR File

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image27.png)

Name it as \"kafkaToMongoDB\" and click Finish

1. Select your project "Example"
2. Select Compile and in-line resources
3. Build and Save

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image28.png)



### 3. ACE Custom Image
#### 3.1 Datasource configuration without SSL
1\. Create "datasources.json" and "Dockerfile" files:

Here is the template for datasources.json 
```
{"mongodb" : 
    {
    "host": "hostname",
    "url":"mongodb+srv://user:password@server.example.com/test",
    "port": 27017,
    "database": "test",
    "user":"username",
    "password":"password",
    "name": "mongodb",
    "connector": "mongodb"
    }
}
```

Dockerfile
```
From ibmcom/ace:latest
RUN . /opt/ibm/ace-11/server/bin/mqsiprofile
ENV PATH "$PATH:/opt/ibm/ace-11/common/jdk/jre/bin:/var/mqsi/extensions/11.0.0/server/bin:/var/mqsi/extensions/11.0.0/bin:/opt/ibm/ace-11/server/bin/mosquitto:/opt/ibm/ace-11/server/bin:/opt/ibm/ace-11/common/node/bin:/opt/ibm/ace-11/tools"
ENV MQSI_WORKPATH "/var/mqsi"
WORKDIR /var/mqsi
RUN npm install loopback-connector-mongodb --save
WORKDIR /home
COPY /datasources.json /home/aceuser/ace-server/config/connectors/loopback/
```


#### 3.2 Datasource configuration with SSL

For connections to a MongoDB instance that require a TLS certificate (such as AWS DocumentDB) upon login you will need to modify the above steps a little bit. You will as well need to create a shell script file. 

```touch add_tls_to_connector.sh```

We're going to use the below shell script to edit the Loopback MongoDB Connector source code to allow for the tls and the tlsCAFile options. By default the sslCA option requires a Buffer object that you can't provide in the datasources.json file. Whereas the tlsCAFile option can be used to provide a string filepath for your certificate. Paste the contents of the below script into the newly created add_tls_to_connector.sh file. 

```
awk '/const validOptionNames/{print;print "'\''tls'\'', '\''tlsCAFile'\'', ";next}1' \
/var/mqsi/node_modules/loopback-connector-mongodb/lib/mongodb.js > \
/var/mqsi/node_modules/loopback-connector-mongodb/lib/temp.js && \
mv /var/mqsi/node_modules/loopback-connector-mongodb/lib/temp.js \
/var/mqsi/node_modules/loopback-connector-mongodb/lib/mongodb.js
```

Make a new custom ACE image with the below Dockerfile. The new commands are to accomodate the need to edit the connector source code.

```
FROM ibmcom/ace:latest
RUN . /opt/ibm/ace-11/server/bin/mqsiprofile
ENV PATH "$PATH:/opt/ibm/ace-11/common/jdk/jre/bin:/var/mqsi/extensions/11.0.0/server/bin:/var/mqsi/extensions/11.0.0/bin:/opt/ibm/ace-11/server/bin/mosquitto:/opt/ibm/ace-11/server/bin:/opt/ibm/ace-11/common/node/bin:/opt/ibm/ace-11/tools"
ENV MQSI_WORKPATH "/var/mqsi"
WORKDIR /var/mqsi
RUN npm install loopback-connector-mongodb --save
WORKDIR /home
COPY /datasources.json /home/aceuser/ace-server/config/connectors/loopback/
COPY /rds-combined-ca-bundle.pem /home/aceuser/ace-server/config/connectors/loopback/
COPY /add_tls_to_connector.sh /home/aceuser/ace-server/config/connectors/loopback/
USER root
RUN chmod u+x /home/aceuser/ace-server/config/connectors/loopback/add_tls_to_connector.sh
RUN /home/aceuser/ace-server/config/connectors/loopback/add_tls_to_connector.sh
```

Edit your datasources.json folder and insert into the <> fields below with your credentials and details.

```
{"mongodb" : {
    "url":"mongodb://<username>:<password>@<mongodburl>:27017/<database_name>?replicaSet=rs0&readPreference=secondaryPreferred",
    "sslValidate":true,
    "useNewUrlParser":true,
    "name": "mongodb",
    "connector": "mongodb",
    "tls": true,
    "tlsCAFile": "/home/aceuser/ace-server/config/connectors/loopback/rds-combined-ca-bundle.pem"
    }
}
```

Note - 
Make sure the newly created shell script, certificate (rds-combined-ca-bundle.pem in this example), datasources.json and this Dockerfile are in the same folder before we proceed to the next step (building and deploying)


#### 3.3 Build and Deploy Image

Make sure datasources.json and Dockerfile are in the same location

2\. Run 
```
docker build . -t ace-mongodb-connector:1.0

```

3\. Push your custom image on openshift 

Login to OpenShift through the command line. Make sure that you are in the correct namespace/project as your deployed ACE Dashboard instance. By default in most cases the ACE Dashboard pods are usually installed in the ace namespace. In my case, I used the ace namespace. To check your current namespace use the "oc project" command. If you aren't in the correct namespace you can change this by using the "oc project ace" command as shown below. Your generated image pull secret and server configuration secrets (which we will generate a bit further down) need to be in the same namespace.

```
oc project ace

docker login $(oc registry info) -u admin -p $(oc whoami -t)

docker tag ace-mongodb-connector:1.0 default-route-openshift-image-registry.apps.ocp43.vg.gse-ocp.net/ace/ace-mongodb-connector:1.0-amd64

docker push default-route-openshift-image-registry.apps.ocp43.vg.gse-ocp.net/ace/ace-mongodb-connector:1.0-amd64
```
Create a secret to access this custom image 
kubectl create secret docker-registry \<secret-name\> --docker-username=$(oc whoami) --docker-password=$(oc whoami -t) --docker-server=image-registry.openshift-image-registry.svc:5000

For example,    
```
oc project ace

kubectl create secret docker-registry internal-reg --docker-username=$(oc whoami) --docker-password=$(oc whoami -t) --docker-server=image-registry.openshift-image-registry.svc:5000
```


### 4. Deploy the bar file on CP4I

Go to your CP4I ace dashboard

1\. Click Create Server -\> Click Add a bar file

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image29.png)

Go to your workspace -\> BARfiles -\> Select kafkaToMongoDB.bar -\>
Click Continue

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image30.png)

2\. Download configuration package on your desktop

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image31.png)

3\. Open the config.tar.gz

Move the java truststore file (es-cert.jks) and api key
(es-api-key.json) here.

4\. Run
```
keytool -list -storetype JKS -keystore es-cert.jks -storepass password -v
```

To show the content of the jks file (password is password)

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image32.png)

Get the alias name from the above output

Use that alias to run the below command:

```
keytool -export -alias es_cacert -file truststoreCert-mykey.crt -keystore es-cert.jks
```

![](./myMediaFolder/media/image33.png)

5\. Open truststorePassword.txt in the config folder. Add your truststore password:

password

Save

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image34.png)

6\. Open setdbparms.txt and add the following:

```
kafka::KAFKA token <api_key>
IntSvr::truststorePass thisispwdfortruststore password
```
- Make sure you keep the text "thisispwdfortruststore" exactly how it is mentioned above. The text "thisispwdfortruststore" is a placeholder. Since we are using truststore certificate we don't need to pass in username so instead we are using random text as a placeholder.

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image35.png)

Copy the api\_key from the es-api-key.json and paste next to token

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image36.png)

7\. Open serverconf.yaml and add following:

```
---
ResourceManagers:
  JVM:
    truststoreType: 'JKS'
    truststoreFile: '/home/aceuser/ace-server/truststore.jks'
    truststorePass: 'IntSvr::truststorePass'

```

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image37.png)

------------------- Important --------------------

RENAME serverconf.yaml to server.conf.yaml

------------------- Important --------------------

8\.
Create a secret
- Go to openshift and copy the login command
- Login to openshift through command line
- Go to the "config" directory through command line

Make sure you are in the same namespace/project as your deployed ACE Dashboard and your custom image.

RUN ./generateSecrets.sh \<secret-name>

For example,
```
oc project ace

./generateSecrets.sh ace-kafka-secret
```

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image38.png)

Go back to CP4I

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image39.png)

Select Toolkit and click Next

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image40.png)

Name: kafka-mongodb

Turn show everything option to ON

Uncheck production usage

Use the custom image for ace only
```
image-registry.openshift-image-registry.svc:5000/ace/ace-mongodb-connector:1.0
```
Since the image is going to be pulled internally in the cluster the endpoint that should be used is image-registry.openshift-image-registry.svc:5000. The registry endpoint that we initially docker logged into and pushed to was for external use (i.e. when docker pushing into it).

Again, please make sure that your custom image and your pull secret were created in the same namespace/project as your deployed ACE Dashboard instance. 

Image pull secret
```
internal-reg
```

![A screenshot of a cell phone Description automatically
generated](./myMediaFolder/media/image41.png)

Name of the secret that contains the server configuration:
ace-kafka-secret

List of key aliases for the keystore: mykey

List of certificate aliases for the truststore: mykey

And leave rest the same

![A screenshot of a computer Description automatically
generated](./myMediaFolder/media/image42.png)

### 5. Test

Open the server named kafkamongodb on ace dashboard and get the URL 

![A screenshot of a computer Description automatically
generated](./myMediaFolder/media/image43.png)

create data.json with the following:
```
{"hello":"world"}
```

```
curl -X POST <url>/producer -d @data.json

Example:
curl -X POST http://kafka-mongodb-http-ace.apps.ocp43.vg.gse-ocp.net/example/v1/producer -d @data.json

```

or you can test using postman
![A screenshot of a computer Description automatically
generated](./myMediaFolder/media/image44.png)


You can see the message in your event streams topic

![A screenshot of a computer Description automatically
generated](./myMediaFolder/media/image45.png)

and finally the same message should be saved in mongodb as well

![A screenshot of a computer Description automatically
generated](./myMediaFolder/media/image46.png)
