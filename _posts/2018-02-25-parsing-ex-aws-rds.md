---
layout: post
title: 'Elixir, AWS, Parsing Describe DB Instances'
categories:
- Elixir
tags: []
---
This is code that I wrote to parse the [ex_aws_rds](https://github.com/ex-aws/ex_aws_rds) describe_db_instances
call. There is no parser in this module currently so parsing the XML is
left as an exercise for anyone using this. Since I had to use the library
in order to get RDS instance lists I figured I'd post what I used in case
someone else wants to convert the XML to something easier to manipulate in
Elixir.

```
  |> SweetXml.xpath(
    ~x".//DBInstance"l,
    allocated_storage: ~x"./AllocatedStorage/text()"s,
    availability_zone: ~x"./AvailabilityZone/text()"s,
    engine_version: ~x"./EngineVersion/text()"s,
    master_username: ~x"./MasterUsername/text()"s,
    instance_create_time: ~x"./InstanceCreateTime/text()"s,
    db_instance_class: ~x"./DBInstanceClass/text()"s,
    monitoring_interval: ~x"./MonitoringInterval/text()"s,
    db_instance_status: ~x"./DBInstanceStatus/text()"s,
    backup_retention_period: ~x"./BackupRetentionPeriod/text()"s,
    ca_certificate_identifier: ~x"./CACertificateIdentifier/text()"s,
    db_instance_port: ~x"./DbInstancePort/text()"s,
    dbi_resource_id: ~x"./DbiResourceId/text()"s,
    preferred_backup_window: ~x"./PreferredBackupWindow/text()"s,
    db_instance_identifier: ~x"./DBInstanceIdentifier/text()"s,
    db_instance_arn: ~x"./DBInstanceArn/text()"s,
    engine: ~x"./Engine/text()"s,
    publicly_accessible: ~x"./PubliclyAccessible/text()"s,
    db_authentication_enabled: ~x"./IAMDatabaseAuthenticationEnabled/text()"s,
    performance_insights_enabled: ~x"./PerformanceInsightsEnabled/text()"s,
    db_name: ~x"./DBName/text()"s,
    multi_az: ~x"./MultiAZ/text()"s,
    storage_encrypted: ~x"./StorageEncrypted/text()"s,
    hosted_zone_id: ~x"./Endpoint/HostedZoneId/text()"s,
    address: ~x"./Endpoint/Address/text()"s,
    port: ~x"./Endpoint/Port/text()"s,
    vpc_id: ~x"./DBSubnetGroup/VpcId/text()"s,
    subnet_group_status: ~x"./DBSubnetGroup/SubnetGroupStatus/text()"s,
    subnet_group_description: ~x"./DBSubnetGroup/DBSubnetGroupDescription/text()"s,
    subnet_group_name: ~x"./DBSubnetGroup/DBSubnetGroupName/text()"s,
    subnets: [
      ~x"./DBSubnetGroup/Subnets/Subnet"l,
      subnet_identifier: ~x"./SubnetIdentifier/text()"s,
      subnet_status: ~x"./SubnetStatus/text()"s,
      subnet_availability_zone: ~x"./SubnetAvailabilityZone/Name/text()"s
    ],
    vpc_security_group_memberships: [
      ~x"./VpcSecurityGroups/VpcSecurityGroupMembership"l,
      vpc_security_group_id: ~x"./VpcSecurityGroupId/text()"s,
      status: ~x"./Status/text()"s
    ],
    license_model: ~x"./LicenseModel/text()"s,
    preferred_maintenance_window: ~x"./PreferredMaintenanceWindow/text()"s,
    storage_type: ~x"./StorageType/text()"s,
    auto_minor_version_upgrade: ~x"./AutoMinorVersionUpgrade/text()"s,
    copy_tags_snapshot: ~x"./CopyTagsToSnapshot/text()"s
  )
```