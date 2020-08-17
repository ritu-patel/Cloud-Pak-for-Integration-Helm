#  How to setup connection from ACE to MQ

## Use case: Save JSON data from a rest api to MQ
## Steps
1.  [Setup MQ](#1-Setup-MQ)
2.  [Create a flow in ACE](#2-ACE-flow)
3.  [Deploy the bar file on CP4I](#3-Deploy-the-bar-file-on-CP4I)
4.  [Test](#4-Test)


## 1. Setup MQ
--------

Select queue manager and the select properties:

![A screenshot of a social media post Description automatically
generated](./images/media/image1.png)

On the Communication tab, find the CHLAUTH Records property and make it
Disabled

![A screenshot of a social media post Description automatically
generated](./images/media/image2.png)

Click on Add widget button on top-right\...

![A screenshot of a cell phone Description automatically
generated](./images/media/image3.png)
\...and add Queues widget

![A screenshot of a social media post Description automatically
generated](./images/media/image4.png)

Click on Create to create new Queue

![A screenshot of a cell phone Description automatically
generated](./images/media/image5.png)

Select Local queue type and and give it a name NEWORDER.MQ

Queue gets successfully created as follows:

![A screenshot of a social media post Description automatically
generated](./images/media/image6.png)

Add Channels widget![A screenshot of a social media post Description
automatically generated](./images/media/image7.png)

**Note:** The next step to create channel can be skipped since it has
been already created.

Click on Create to create a new channel **ACE.TO.MQ**

![A screenshot of a cell phone Description automatically
generated](./images/media/image8.png)

\... call it **ACE.TO.MQ**, select **Server-connection** channel type

![A screenshot of a social media post Description automatically
generated](./images/media/image9.png)

Select the channel and click Properties

![A screenshot of a cell phone Description automatically
generated](./images/media/image10.png)

On the **MCA tab**, for **MCA User ID** specify **mqm**

![A screenshot of a cell phone Description automatically
generated](./images/media/image11.png)

Click Add widget again and select Authentication Information widget

![A screenshot of a social media post Description automatically
generated](./images/media/image12.png)

Click on the cogwheel to configure the widget

![](./images/media/image13.png)

\... and select **System objects: Show**![A screenshot of a cell phone
Description automatically
generated](./images/media/image14.png)

Select SYSTEM.DEFAULT.AUTHINFO.IDPWOS and then click Properties

![A screenshot of a cell phone Description automatically
generated](./images/media/image15.png)

On the User ID + password tab, for both Local connections and for Client
connections specify None

![A screenshot of a social media post Description automatically
generated](./images/media/image16.png)

-   Select Save and Close
-   Select queue manager again and click on \"three dots\" icon, the
    select \"Refresh security\"

![A screenshot of a social media post Description automatically
generated](./images/media/image17.png)

Click on Connection authentication

![A screenshot of a social media post Description automatically
generated](./images/media/image18.png)

**Note:** You have now completed the step to create the MQ
queue **NEWORDER.MQ** & the MQ channel **ACE.TO.MQ** and configured
security.

Confirm MQ Listener port

-   Still in MQ Console, add Listeners widget

![A screenshot of a social media post Description automatically
generated](./images/media/image19.png)

Configure widget to see system objects

![A screenshot of a cell phone Description automatically
generated](./images/media/image20.png)

Select System objects: Show

![A screenshot of a cell phone Description automatically
generated](./images/media/image21.png)

Check the value of SYSTEM.LISTENER.TCP.1 (default is 1414)

![A screenshot of a cell phone Description automatically
generated](./images/media/image22.png)

Navigate to OpenShift console, select **mq** project and the select
Networking \> Services![A screenshot of a cell phone Description
automatically
generated](./images/media/image42.png)

Select the service whose name starts with the queue manager name and has
extension \'-ibm-mq\'. For example, the service in this case is
mq-01-assets-op-ibm-mq

Note:
-   We can access the listener from the outside of the cluster using the
    cluster proxy host name and this port by creating a route.
-   However, if we are accessing the mq inside the cluster, we can use
    \<service\_name\>.mq.svc.cluster.local for the host name and target
    listener port which is 1414. For example, mq-01-assets-op-ibm-mq.mq.svc.cluster.local

**Note:** You have now completed the step to inspect the MQ connection
information. It will be used in the next for configuring MQ Client.

From MQ that we just created, we will need the following information for
ACE

In this case,

-   Hostname: mq-01-assets-op-ibm-mq.mq.svc.cluster.local

-   Port: 1414

-   Queue name: NEWORDER.MQ

-   Channel Name: ACE.TO.MQ

-   Queue Manger: mq01assetsop

## 2. ACE flow
----------------

Open ACE toolkit

File -\> New -\> REST API

![A screenshot of a cell phone Description automatically
generated](./images/media/image26.png)
Name it MQdemo and click Finish

![A screenshot of a cell phone Description automatically
generated](./images/media/image27.png)

Add Resources by clicking the + symbol

Select POST method

Name the path: /msg

![A screenshot of a computer Description automatically
generated](./images/media/image28.png)

File -\> Save

Now Add a subflow under the /msg resources that you just created

![A screenshot of a cell phone Description automatically
generated](./images/media/image29.png)

Add nodes like below

![](./images/media/image30.png)

Add Compute configuration

![A screenshot of a cell phone Description automatically
generated](./images/media/image31.png)

Double click on compute node and add the following code:

Compute node - Only saves the accountid and orderid
```
    CREATE COMPUTE MODULE New_Order_Compute
        CREATE FUNCTION Main() RETURNS BOOLEAN
        BEGIN
            Set OutputRoot.JSON.Data.accountid = InputRoot.JSON.Data.accountid;
            Set OutputRoot.JSON.Data.orderid = SUBSTRING(CAST(CURRENT_TIMESTAMP AS CHARACTER) AFTER '.' FOR 6);
            RETURN TRUE;
        END;
        CREATE PROCEDURE CopyMessageHeaders() BEGIN
            DECLARE I INTEGER 1;
            DECLARE J INTEGER;
            SET J = CARDINALITY(InputRoot.*[]);
            WHILE I < J DO
                SET OutputRoot.*[I] = InputRoot.*[I];
                SET I = I + 1;
            END WHILE;
        END;
        CREATE PROCEDURE CopyEntireMessage() BEGIN
            SET OutputRoot = InputRoot;
        END;
    END MODULE;
```
Add HTTP header node properties

![A screenshot of a social media post Description automatically
generated](./images/media/image32.png)

Add MQ output node properties from MQ that we just created

In this case,

-   Hostname: mq-01-assets-op-ibm-mq.mq.svc.cluster.local

-   Port: 1414

-   Queue name: NEWORDER.MQ

-   Channel Name: ACE.TO.MQ

-   Queue Manger: mq01assetsop

![A screenshot of a cell phone Description automatically
generated](./images/media/image33.png)

![A screenshot of a cell phone Description automatically
generated](./images/media/image34.png)

Save it as a bar file

To save it as a bar file:

File -> New -> Bar File

Name it as: MQdemo

Select the project and the necessary options to build and save the bar

![A screenshot of a cell phone Description automatically
generated](./images/media/image35.png)

File -> Save

## 3. Deploy the bar file on CP4I
-----------------------------------

Go to ace dashboard and click on "Create Server"

![A screenshot of a cell phone Description automatically
generated](./images/media/image36.png)

Upload the bar file: MQdemo

![A screenshot of a cell phone Description automatically
generated](./images/media/image37.png)

Select Toolkit and hit Next

![A screenshot of a cell phone Description automatically
generated](./images/media/image38.png)

Turn "Show everything" to ON

Now fill out the helm chart

-   **Helm release name**: mq-demo
-   Uncheck Production usage if its not production
-   **Image**: App Connect Enterprise with MQ client
-   **Image pull secret**: cp-entitlement (the secret for your image)

Click Create

![A screenshot of a social media post Description automatically
generated](./images/media/image39.png)

## 4. Test
------------

Go to ACE dashboard

![A screenshot of a cell phone Description automatically
generated](./images/media/image40.png)

Copy the REST API Base URL

```
{
    "accountid": "B-10000",
    "order": {
        "orderDate": "2004-01-19T04:25:25.938Z",
        "contractId": "00000100",
        "orderDetails": [
            {
                "lineItemNumber": 1,
                "productId": "AJ1-05",
                "quantity": "20"
            }
        ]
    }
}
```

Go to postman or curl the post request\
![A screenshot of a social media post Description automatically
generated](./images/media/image41.png)

You should be able to see a message in MQ
