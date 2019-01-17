# Controlling Which Auto Scaling Instances Terminate During Scale In<a name="as-instance-termination"></a>

With each Auto Scaling group, you control when it adds instances \(referred to as *scaling out*\) or remove instances \(referred to as *scaling in*\) from your network architecture\. You can scale the size of your group manually by attaching and detaching instances, or you can automate the process through the use of a scaling policy\.

When you configure automatic scale in, you must set up a termination policy for the Auto Scaling group to use to choose which instances to terminate first\.

You can also use instance protection to prevent specific instances from being terminated during automatic scale in\.

**Note**  
The information in this section about maximizing usage to the hour is currently applicable to instances running Microsoft Windows or Linux distributions that have an hourly charge\. If your Auto Scaling group uses Amazon Linux or Ubuntu, your EC2 usage is billed in one\-second increments\. For more information about per\-second billing, see [Amazon EC2 Pricing](https://aws.amazon.com/ec2/pricing/)\.

**Topics**
+ [Default Termination Policy](#default-termination-policy)
+ [Customizing the Termination Policy](#custom-termination-policy)
+ [Instance Protection](#instance-protection)

## Default Termination Policy<a name="default-termination-policy"></a>

The default termination policy is designed to help ensure that your instances span Availability Zones evenly for high availability\. 

With the default termination policy, the behavior of the Auto Scaling group is as follows:

1. Determine which Availability Zone\(s\) have the most instances and at least one instance that is not protected from scale in\. 

1. If there are multiple unprotected instances to choose from in the Availability Zone\(s\) with the most instances: 

   1. For Auto Scaling groups configured to [launch instances across pricing options and instance types](AutoScalingGroup.md#asg-purchase-options):

      Determine which instance to terminate so as to align the remaining instances to the allocation strategies for On\-Demand and Spot Instances, your current selection of instance types, and distribution across your N lowest priced Spot pools\. 

   1. If there are multiple instances to choose from, determine whether any of these instances use the oldest launch configuration or template\. If there is one such instance, terminate it\. 

   1. For Auto Scaling groups that use a launch configuration: 

      Determine whether any of these instances use the oldest launch configuration\. If there is one such instance, terminate it\. 

1. If there are multiple unprotected instances to terminate based on the above criteria, determine which instances are closest to the next billing hour\. \(This helps you maximize the use of your instances that have an hourly charge\.\) If there is one such instance, terminate it\. 

1. If there are multiple unprotected instances closest to the next billing hour, choose one of these instances at random\.

**Example**  
Consider an Auto Scaling group that uses a launch configuration\. It has one instance type, two Availability Zones, a desired capacity of two instances, and scaling policies that increase and decrease the number of instances by 1 when certain thresholds are met\. The two instances in this group are distributed as follows\.

![\[A basic Auto Scaling group.\]](http://docs.aws.amazon.com/autoscaling/ec2/userguide/images/termination-policy-default-diagram.png)

When the threshold for the scale\-out policy is met, the policy takes effect and the Auto Scaling group launches a new instance\. The Auto Scaling group now has three instances, distributed as follows\.

![\[An Auto Scaling group after a scaling action occurs.\]](http://docs.aws.amazon.com/autoscaling/ec2/userguide/images/termination-policy-default-2-diagram.png)

When the threshold for the scale\-in policy is met, the policy takes effect and the Auto Scaling group terminates one of the instances\. If you did not assign a specific termination policy to the group, it uses the default termination policy\. It selects the Availability Zone with two instances, and terminates the instance launched from the oldest launch configuration\. If the instances were launched from the same launch configuration, then the Auto Scaling group selects the instance that is closest to the next billing hour and terminates it\.

For more information about high availability with Amazon EC2 Auto Scaling, see [Distributing Instances Across Availability Zones](auto-scaling-benefits.md#arch-AutoScalingMultiAZ)\.

## Customizing the Termination Policy<a name="custom-termination-policy"></a>

The default termination policy assigned to an Auto Scaling group is typically sufficient for most situations\. However, you have the option of replacing the default policy with a customized one\.

When you customize the termination policy, if one Availability Zone has more instances than the other Availability Zones that are used by the group, then your termination policy is applied to the instances from the imbalanced Availability Zone\. If the Availability Zones used by the group are balanced, then the termination policy is applied across all the Availability Zones for the group\.  Additionally, if you are using an Auto Scaling Group with multiple instance types and purchase options, the Spot and On Demand market options will be balanced before applying your termination policy\.  The group will pick which of these two types of instances it will terminate and then pick the Availability Zone with the most instances of that type to apply your termination policy to\.

Amazon EC2 Auto Scaling supports the following custom termination policies:
+ `OldestInstance`\. Terminate the oldest instance in the group\. This option is useful when you're upgrading the instances in the Auto Scaling group to a new EC2 instance type\. You can gradually replace instances of the old type with instances of the new type\.
+ `NewestInstance`\. Terminate the newest instance in the group\. This policy is useful when you're testing a new launch configuration but don't want to keep it in production\.
+ `OldestLaunchConfiguration`\. Terminate instances that have the oldest launch configuration\. This policy is useful when you're updating a group and phasing out the instances from a previous configuration\.
+ `ClosestToNextInstanceHour`\. Terminate instances that are closest to the next billing hour\. This policy helps you maximize the use of your instances and manage your Amazon EC2 usage costs\.
+ `Default`\. Terminate instances according to the default termination policy\. This policy is useful when you have more than one scaling policy for the group\.
+ `OldestLaunchTemplate`\. Terminate instances that have the oldest launch template: first the non\-current launch template followed by the oldest version of the current launch template\. This policy is useful when you're updating a group and phasing out the instances from a previous configuration\.
+ `AllocationStrategy`\. Terminate instances in the Auto Scaling group to align the instance distribution to the allocation strategies for On\-Demand and Spot Instances, your current selection of instance types, and distribution across your N lowest priced Spot pools\. 

**To customize a termination policy using the console**

1. Open the Amazon EC2 console at [https://console\.aws\.amazon\.com/ec2/](https://console.aws.amazon.com/ec2/)\.

1. On the navigation pane, choose **Auto Scaling Groups**\.

1. Select the Auto Scaling group\.

1. For **Actions**, choose **Edit**\.

1. On the **Details** tab, locate **Termination Policies**\. Choose one or more termination policies\. If you choose multiple policies, list them in the order in which they should apply\. If you use the **Default** policy, make it the last one in the list\.

1. Choose **Save**\.

**To customize a termination policy using the AWS CLI**  
Use one of the following commands:
+ [https://docs.aws.amazon.com/cli/latest/reference/autoscaling/create-auto-scaling-group.html](https://docs.aws.amazon.com/cli/latest/reference/autoscaling/create-auto-scaling-group.html)
+ [https://docs.aws.amazon.com/cli/latest/reference/autoscaling/update-auto-scaling-group.html](https://docs.aws.amazon.com/cli/latest/reference/autoscaling/update-auto-scaling-group.html)

You can use these policies individually, or combine them into a list of policies\. For example, use the following command to update an Auto Scaling group to use the `OldestLaunchConfiguration` policy first and then use the `ClosestToNextInstanceHour` policy:

```
aws autoscaling update-auto-scaling-group --auto-scaling-group-name my-asg --termination-policies "OldestLaunchConfiguration" "ClosestToNextInstanceHour"
```

If you use the `Default` termination policy, make it the last one in the list of termination policies\. For example, `--termination-policies "OldestLaunchConfiguration" "Default"`\.

## Instance Protection<a name="instance-protection"></a>

To control whether an Auto Scaling group can terminate a particular instance when scaling in, use instance protection\. You can enable the instance protection setting on an Auto Scaling group or an individual Auto Scaling instance\. When the Auto Scaling group launches an instance, it inherits the instance protection setting of the Auto Scaling group\. You can change the instance protection setting for an Auto Scaling group or an Auto Scaling instance at any time\.

Instance protection starts when the instance state is `InService`\. If you detach an instance that is protected from termination, its instance protection setting is lost\. When you attach the instance to the group again, it inherits the current instance protection setting of the group\.

If all instances in an Auto Scaling group are protected from termination during scale in and a scale\-in event occurs, its desired capacity is decremented\. However, the Auto Scaling group can't terminate the required number of instances until their instance protection settings are disabled\.

Instance protection does not protect Auto Scaling instances from the following:
+ Manual termination through the Amazon EC2 console, the `terminate-instances` command, or the `TerminateInstances` action\. To protect Auto Scaling instances from manual termination, enable termination protection\. For more information, see [Enabling Termination Protection](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html#Using_ChangingDisableAPITermination) in the *Amazon EC2 User Guide for Linux Instances*\.
+ Health check replacement if the instance fails health checks\.
+ Spot Instance interruption\.

**Topics**
+ [Enable Instance Protection for a Group](#instance-protection-group)
+ [Modify the Instance Protection Setting for a Group](#instance-protection-modify)
+ [Modify the Instance Protection Setting for an Instance](#instance-protection-instance)

### Enable Instance Protection for a Group<a name="instance-protection-group"></a>

You can enable instance protection when you create an Auto Scaling group\. By default, instance protection is disabled\.

**To enable instance protection using the console**  
When you create the Auto Scaling group, on the **Configure Auto Scaling group details** page, under **Advanced Details**, select the `Protect From Scale In` option from **Instance Protection**\.

![\[Enable Instance Protection using the wizard.\]](http://docs.aws.amazon.com/autoscaling/ec2/userguide/images/as-enable-instance-protection.png)

**To enable instance protection using the AWS CLI**  
Use the following [https://docs.aws.amazon.com/cli/latest/reference/autoscaling/create-auto-scaling-group.html](https://docs.aws.amazon.com/cli/latest/reference/autoscaling/create-auto-scaling-group.html) command to enable instance protection:

```
aws autoscaling create-auto-scaling-group --auto-scaling-group-name my-asg --new-instances-protected-from-scale-in ...
```

### Modify the Instance Protection Setting for a Group<a name="instance-protection-modify"></a>

You can enable or disable the instance protection setting for an Auto Scaling group\.

**To change the instance protection setting for a group using the console**

1. Open the Amazon EC2 console at [https://console\.aws\.amazon\.com/ec2/](https://console.aws.amazon.com/ec2/)\.

1. On the navigation pane, choose **Auto Scaling Groups**\.

1. Select the Auto Scaling group\.

1. On the **Details** tab, choose **Edit**\.

1. For **Instance Protection**, select **Protect From Scale In**\.  
![\[View Instance Protection on the Details tab.\]](http://docs.aws.amazon.com/autoscaling/ec2/userguide/images/as-modify-instance-protection.png)

1. Choose **Save**\.

**To change the instance protection setting for a group using the AWS CLI**  
Use the following [https://docs.aws.amazon.com/cli/latest/reference/autoscaling/update-auto-scaling-group.html](https://docs.aws.amazon.com/cli/latest/reference/autoscaling/update-auto-scaling-group.html) command to enable instance protection for the specified Auto Scaling group:

```
aws autoscaling update-auto-scaling-group --auto-scaling-group-name my-asg --new-instances-protected-from-scale-in
```

Use the following command to disable instance protection for the specified group:

```
aws autoscaling update-auto-scaling-group --auto-scaling-group-name my-asg --no-new-instances-protected-from-scale-in
```

### Modify the Instance Protection Setting for an Instance<a name="instance-protection-instance"></a>

By default, an instance gets its instance protection setting from its Auto Scaling group\. However, you can enable or disable instance protection for an instance at any time\.

**To change the instance protection setting for an instance using the console**

1. Open the Amazon EC2 console at [https://console\.aws\.amazon\.com/ec2/](https://console.aws.amazon.com/ec2/)\.

1. On the navigation pane, choose **Auto Scaling Groups**\.

1. Select the Auto Scaling group\.

1. On the **Instances** tab, select the instance\.

1. To enable instance protection, choose **Actions**, **Instance Protection**, **Set Scale In Protection**\. When prompted, choose **Set Scale In Protection**\.

1. To disable instance protection, choose **Actions**, **Instance Protection**, **Remove Scale In Protection**\. When prompted, choose **Remove Scale In Protection**\.

**To change the instance protection setting for an instance using the AWS CLI**  
Use the following [https://docs.aws.amazon.com/cli/latest/reference/autoscaling/set-instance-protection.html](https://docs.aws.amazon.com/cli/latest/reference/autoscaling/set-instance-protection.html) command to enable instance protection for the specified instance:

```
aws autoscaling set-instance-protection --instance-ids i-5f2e8a0d --auto-scaling-group-name my-asg --protected-from-scale-in
```

Use the following command to disable instance protection for the specified instance:

```
aws autoscaling set-instance-protection --instance-ids i-5f2e8a0d --auto-scaling-group-name my-asg --no-protected-from-scale-in
```
