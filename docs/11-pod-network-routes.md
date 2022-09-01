# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

Print the internal IP address and Pod CIDR range for each worker instance, then create a route for each worker instance in the `kubernetes-the-hard-way` VPC network.:

```
for instance in worker-0 worker-1 worker-2; do
  instance_id_ip="$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress]')"
  instance_id="$(echo "${instance_id_ip}" | cut -f1)"
  instance_ip="$(echo "${instance_id_ip}" | cut -f2)"
  pod_cidr="$(aws ec2 describe-instance-attribute \
    --instance-id "${instance_id}" \
    --attribute userData \
    --output text --query 'UserData.Value' \
    | base64 --decode | tr "|" "\n" | grep "^pod-cidr" | cut -d'=' -f2)"
  echo "${instance_ip} ${pod_cidr}"

  aws ec2 create-route \
    --route-table-id "${ROUTETABLE}" \
    --destination-cidr-block "${pod_cidr}" \
    --instance-id "${instance_id}"
done
```

> output

```
done
10.0.1.20 10.200.0.0/24
{
    "Return": true
}
10.0.1.21 10.200.1.0/24
{
    "Return": true
}
10.0.1.22 10.200.2.0/24
{
    "Return": true
}
```

## Validate

Create network routes for each worker instance:

```
aws ec2 describe-route-tables \
  --route-table-ids "${ROUTETABLE}" \
  --query 'RouteTables[].Routes' \
  --output table
```

> output

```
----------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                DescribeRouteTables                                                               |
+----------------------+------------------------+----------------------+------------------+------------------------+--------------------+----------+
| DestinationCidrBlock |       GatewayId        |     InstanceId       | InstanceOwnerId  |  NetworkInterfaceId    |      Origin        |  State   |
+----------------------+------------------------+----------------------+------------------+------------------------+--------------------+----------+
|  10.64.0.0/24        |  local                 |                      |                  |                        |  CreateRouteTable  |  active  |
|  10.200.0.0/24       |                        |  i-02d6ee530ee3a7c90 |  677745302030    |  eni-0d22bf412657b2c0c |  CreateRoute       |  active  |
|  10.200.1.0/24       |                        |  i-085259cb120920c78 |  677745302030    |  eni-0ddf6ae4c5ff0b2f7 |  CreateRoute       |  active  |
|  10.200.2.0/24       |                        |  i-0292bfd1d6b63920b |  677745302030    |  eni-0a606e9df6a656482 |  CreateRoute       |  active  |
|  10.0.0.0/16         |  local                 |                      |                  |                        |  CreateRouteTable  |  active  |
|  0.0.0.0/0           |  igw-01eef992992af026b |                      |                  |                        |  CreateRoute       |  active  |
+----------------------+------------------------+----------------------+------------------+------------------------+--------------------+----------+
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
