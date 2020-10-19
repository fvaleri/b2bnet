# Broker-to-broker network
Broker-to-broker qpid-router network on OpenShift connecting two data centers
(OpenShift 4.5.14, AMQ Broker 7.7, AMQ Interconnect 1.9).

In this example the two data centers are represented by two namespaces called `dc1` and `dc2`.
In a real cross-data data center setup you would need to add TLS authentication for router connections.
All routers reconnect in case of network failures and resend any in-flight message.
```sh
# Topology (each connection has two links in opposite directions)
[dc1-broker] <-- (dc1-router) --> (dc1-gateway) <-- (dc2-router) --> [dc2-broker]

# DC1 routes (clients can produce to b2b.dc2.incoming and consume from b2b.dc2.outgoing)
[dc1-broker] --> (dc1-router) --> (dc1-gateway) --> ..network.. --> (dc2-router) --> [dc2-broker (address: b2b.dc2.incoming)]
[dc1-broker] <-- (dc1-router) <-- (dc1-gateway) <-- ..network.. <-- (dc2-router) <-- [dc2-broker (address: b2b.dc2.outgoing)]

# DC2 routes (clients can produce to b2b.dc1.incoming and consume from b2b.dc1.outgoing)
[dc2-broker] --> (dc2-router) --> ..network.. --> (dc1-gateway) --> (dc1-router) --> [dc1-broker (address: b2b.dc1.incoming)]
[dc2-broker] <-- (dc2-router) <-- ..network.. <-- (dc1-gateway) <-- (dc1-router) <-- [dc1-broker (address: b2b.dc1.outgoing)]
```

## Configuration
Login as `cluster-admin` user to execute the following commands.
```sh
# deploy the cluster-wide certificate manager operator
oc create -f olm/router/cm-sub.yaml

# repeat the following steps for each data center
USERNAME="developer"
NAMESPACE="dc1"
TMP="/tmp/ocp"

rm -rf $TMP && mkdir -p $TMP
oc new-project $NAMESPACE
oc adm policy add-role-to-user admin $USERNAME

# create the local operator group
cp olm/og.yaml $TMP
sed -i .bk "s/changeit/$NAMESPACE/g" $TMP/og.yaml
oc create -f $TMP/og.yaml
oc get og

# deploy the router namespaced operator
cp olm/router/ir-sub.yaml $TMP
sed -i .bk "s/changeit/$NAMESPACE/g" $TMP/ir-sub.yaml
oc create -f $TMP/ir-sub.yaml

# deploy the broker namespaced operator
cp olm/broker/sub.yaml $TMP
sed -i .bk "s/changeit/$NAMESPACE/g" $TMP/sub.yaml
oc create -f $TMP/sub.yaml

# check deployed components
oc get pods -n openshift-operators
oc get og -n openshift-operators
oc get pods

# deploy the gateway router (dc1 only, skip for dc2)
oc create -f $NAMESPACE/gateway.yaml

# deploy the local broker
oc create -f $NAMESPACE/broker.yaml
# wait for the broker to be running before adding custom configuration
oc create configmap broker-config --from-file=$NAMESPACE/broker.xml
oc set volume statefulset $NAMESPACE-broker-ss --add --overwrite \
    --name=broker-config \
    --mount-path=/opt/amq/conf/broker.xml \
    --sub-path=broker.xml \
    --source='{"configMap":{"name":"broker-config"}}'
oc delete pods -l ActiveMQArtemis=$NAMESPACE-broker

# deploy the router
oc create -f $NAMESPACE/router.yaml

# web console access (user: guest@$NAMESPACE-router)
echo "https://$(oc get route $NAMESPACE-router-8080 -o=jsonpath='{.status.ingress[0].host}{"\n"}')"
echo $(oc get secret $NAMESPACE-router-users -o yaml -n $NAMESPACE | grep " guest: " | awk '{print $2}' | base64 -d)

# external client access (connection URL and truststore)
echo "amqps://$NAMESPACE-router-5671-$NAMESPACE.apps-crc.testing:443?transport.verifyHost=false"
oc extract secret/$NAMESPACE-router-default-credentials -n $NAMESPACE --keys=ca.crt --to=- > ca.crt
keytool -import -noprompt -trustcacerts -alias root -file ca.crt -keystore $NAMESPACE-client.ts -storepass changeit
```

## Tests
In case of network issues you may have released count greater than zero,
which are messages that did not reach the destination (`del = acc + rel`).
```sh
LOCAL_DC="dc1"
REMOTE_DC="dc2"
oc project $LOCAL_DC

ROUTER_POD=$(oc get pods | grep $LOCAL_DC-router | grep Running | cut -d " " -f1)
ROUTER_URL="tcp://$LOCAL_DC-router:5672"
BROKER_POD="$LOCAL_DC-broker-ss-0"
BROKER_URL="tcp://$LOCAL_DC-broker-hdls-svc:61616"

# broker/router stats
oc exec -it $BROKER_POD -- amq-broker/bin/artemis queue stat --url $BROKER_URL
oc exec -it $ROUTER_POD -- qdstat -l

# produce messages directly to remote-broker using local-router
oc exec -it $BROKER_POD -- amq-broker/bin/artemis producer \
    --url $ROUTER_URL --destination queue://b2b.$REMOTE_DC.incoming \
    --protocol amqp --message test --message-count 10 #--sleep 1000

# produce messages for remote-consumers to local-broker
oc exec -it $BROKER_POD -- amq-broker/bin/artemis producer \
    --url $BROKER_URL --destination queue://b2b.$LOCAL_DC.outgoing \
    --message test --message-count 10

# consume messages directly from remote-broker using local-router
oc exec -it $BROKER_POD -- amq-broker/bin/artemis consumer \
    --url $ROUTER_URL --destination queue://b2b.$REMOTE_DC.outgoing \
    --protocol amqp --message-count 10

# consume messages sent by remote-producers from local-broker
oc exec -it $BROKER_POD -- amq-broker/bin/artemis consumer \
    --url $BROKER_URL --destination queue://b2b.$LOCAL_DC.incoming \
    --message-count 10

###
### Initial statistics.
###

|NAME             |ADDRESS          |CONSUMER_COUNT |MESSAGE_COUNT |MESSAGES_ADDED |DELIVERING_COUNT |MESSAGES_ACKED
# dc1-broker
|b2b.dc1.incoming |b2b.dc1.incoming |0              |0             |0              |0                |0
|b2b.dc1.outgoing |b2b.dc1.outgoing |1              |0             |0              |0                |0
# dc2-broker
|b2b.dc2.incoming |b2b.dc2.incoming |0              |0             |0              |0                |0
|b2b.dc2.outgoing |b2b.dc2.outgoing |1              |0             |0              |0                |0

type          dir  conn id  id  addr              phs  cap  undel  deliv  acc   rej  rel  mod  delay  rate  stuck  cred
========================================================================================================================
# dc1-router
endpoint      out  3        26  b2b.dc1.incoming  0    250  0      0      0     0    0    0    0      0     0      1000
endpoint      in   3        27  b2b.dc1.outgoing  1    250  0      0      0     0    0    0    0      0     0      0
# dc2-router
endpoint      out  3        26  b2b.dc2.incoming  0    250  0      0      0     0    0    0    0      0     0      1000
endpoint      in   3        27  b2b.dc2.outgoing  1    250  0      0      0     0    0    0    0      0     0      0

###
### Send and cosume 10 messages in both directions.
###

|NAME             |ADDRESS          |CONSUMER_COUNT |MESSAGE_COUNT |MESSAGES_ADDED |DELIVERING_COUNT |MESSAGES_ACKED
# dc1-broker
|b2b.dc1.incoming |b2b.dc1.incoming |0              |0             |10             |0                |10
|b2b.dc1.outgoing |b2b.dc1.outgoing |1              |0             |10             |0                |10
# dc2-broker
|b2b.dc2.incoming |b2b.dc2.incoming |0              |0             |10             |0                |10
|b2b.dc2.outgoing |b2b.dc2.outgoing |1              |0             |10             |0                |10

type          dir  conn id  id  addr              phs  cap  undel  deliv  acc   rej  rel  mod  delay  rate  stuck  cred
========================================================================================================================
# dc1-router
endpoint      out  750      41  b2b.dc1.incoming  0    250  0      10     10    0    0    0    0      0     0      990
endpoint      in   750      42  b2b.dc1.outgoing  1    250  0      10     10    0    0    0    0      0     0      250
# dc2-router
endpoint      out  751      33  b2b.dc2.incoming  0    250  0      10     10    0    0    0    0      0     0      990
endpoint      in   751      34  b2b.dc2.outgoing  1    250  0      10     10    0    0    0    0      0     0      250
```
