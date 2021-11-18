   ### STEPS TO CREATE CLOUD FORMATION TEMPLATE (CFT) AND CI/CD USING GITHUB AGENT 
                      
  
  
  ![alt text](https://github.com/tanersa/albasg-cft/blob/master/diagram/CloudFormationConcept.png)
  
  Our main goal is to have immutable infrastructure, so we build our VPC in order to achieve network isolation. Therefore, the highest priority in Software Development Life Cycle is the SECURITY! Cloudformation will provison our infrastructure. Eventually, CFT will be used to create a stack. We usually use json/ymal format for creating CFT. 
  
- At the very beginning of CFT, we define our CFT name as a description by following naming convention. Then, we would have keyname with logical id as well as the type as below:
  
      "AWSTemplateFormatVersion": "2010-09-09",
      "Description": "This CFT provisions AWS VPC, ALB and ASG for Wordpress HA",
      "Parameters": 
          "WebServerKeyName": 
              "Description": "This key will be used to SSH to our EC2 instances",
              "Type": "AWS::EC2::KeyPair::KeyName"
       
       
       
  Now, lets start adding our resources...
  
- As a first resource, we can add our VPC in infrastructure which will allow us to achieve network isolation. For VPC, all the IP addresses should be on specific range so we put our CIDR block for that. These IP addresses will help the computers to communicate to each other. Tags could be added to name our VPC. 
  
      "Resources" : {
        "VPC":{
            "Type" : "AWS::EC2::VPC",
            "Properties" : {
                "CidrBlock" : "10.0.0.0/16",
                "EnableDnsHostnames" : true,
                "EnableDnsSupport" : true,
                "InstanceTenancy" : "default",
                "Tags" : [ {"Key" : "Name", "Value":"BlogVPC"} ]
            }
        },
        
 - Next step is to build our subnets (sub-networks) inside the VPC. In AWS, we go by the regions either US-East-1A,  US-East-1B, Dublin, Tokyo, China , etc. We deploy our resources in regions inside this infrastructure. VPC is our region and inside this network we have subnetworks also called **Availability Zones** (AZs). There are public subnets and private subnets in this same VPC. We would have minimum two AZs in each region in order to achieve **High Availability** (HA). If something goes wrong within a region, we can still protect our data so we don't loose it. In any case, we should be able to replicate the data in other AZ within the region.

        Example:
        "DMZ1Public" : {
              "Type" : "AWS::EC2::Subnet",
              "Properties" : {
                  "AvailabilityZone" : {"Fn::Select" : ["0",{"Fn::GetAZs":""}]},
                  "CidrBlock" : "10.0.0.0/24",
                  "MapPublicIpOnLaunch" : true,
                  "Tags" : [ {"Key" : "Name", "Value" : "DMZ1Public"} ],
                  "VpcId" : {"Ref" : "VPC"}
              }
          },
          "AppLayer1Private" : {
            "Type" : "AWS::EC2::Subnet",
            "Properties" : {
                "AvailabilityZone" : {"Fn::Select" : ["0",{"Fn::GetAZs":""}]},
                "CidrBlock" : "10.0.10.0/24",
                "MapPublicIpOnLaunch" : false,
                "Tags" : [ {"Key" : "Name", "Value" : "AppLayer1Private"} ],
                "VpcId" : {"Ref" : "VPC"}
            }
        },
          
- We still build our network and continue with adding Intenet Gateway (IGW) for our VPC. This will allow us to reach out to the internet.

          "InternetGateway" : {
            "Type" : "AWS::EC2::InternetGateway",
            "Properties" : {
                "Tags" : [ {"Key" : "Name", "Value" : "BlogIGW"} ]
            }
        },
        
 - To achieve internet connectivity, there is another component called Internet Gateway Attachment.

          "InternetGatewayAttachment" : {
            "Type" : "AWS::EC2::VPCGatewayAttachment",
            "Properties" : {
                "InternetGatewayId" : {"Ref" : "InternetGateway"},
                "VpcId" : {"Ref" : "VPC"}
            }
        },
        
- Another piece of our VPC network is Route Tables (RTs). We have Public RTs and Private RTs to add our network as below...

          "PublicRT" : {
            "Type" : "AWS::EC2::RouteTable",
            "Properties" : {
                "Tags" : [ {"Key" : "Name", "Value" : "Public-RT"} ],
                "VpcId" : {"Ref" : "VPC"}
            }
        },
        "PublicRT" : {
            "Type" : "AWS::EC2::RouteTable",
            "Properties" : {
                "Tags" : [ {"Key" : "Name", "Value" : "Public-RT"} ],
                "VpcId" : {"Ref" : "VPC"}
            }
        },
        
- Then for again reaching out to internet, we add a route to our Public RT.

        "PublicRTRoute" : {
            "Type" : "AWS::EC2::Route",
            "DependsOn" : "InternetGatewayAttachment",
            "Properties" : {
                "DestinationCidrBlock" : "0.0.0.0/0",
                "GatewayId" : {"Ref" :"InternetGateway"},
                "RouteTableId" : {"Ref" : "PublicRT"}
            }
        },
        
- Now, lets associate our subnets to our public and private RTs.

        "PublicRTDMZ1Association" : {
            "Type" : "AWS::EC2::SubnetRouteTableAssociation",
            "Properties" : {
                "RouteTableId" : {"Ref" : "PublicRT"},
                "SubnetId" : {"Ref" : "DMZ1Public"}
            }
        },
        "AppLayer1PrivateSubnetAssociation" : {
            "Type" : "AWS::EC2::SubnetRouteTableAssociation",
            "Properties" : {
                "RouteTableId" : {"Ref" : "PrivateRT"},
                "SubnetId" : {"Ref" : "AppLayer1Private"}
            }
        },
        
- We also need NAT Gatweays which will allow us to go to private subnets from public subnets and have internet connectivity in a secure way.

        "DMZ2NatGW" : {
            "Type" : "AWS::EC2::NatGateway",
            "Properties" : {
                "AllocationId" : { "Fn::GetAtt" : ["NatEIP", "AllocationId"]},
                "SubnetId" : {"Ref" :"DMZ2Public"},
                "Tags" : [ {"Key" : "Name",  "Value" :"NAT-GATEWAY"} ]
            }
        },
        
- To have NAT Gateway (NATGW), getting Elastic IP adresses is mandatory. Then we add Elastic IPs to our NATGW. 

        "NatEIP" : {
            "Type" : "AWS::EC2::EIP",
            "DependsOn" : "InternetGatewayAttachment",
            "Properties" : {
                "Domain" : "vpc",
                "Tags" : [ {"Key" :"Name", "Value" : "Elastic-IP"} ]
            }
        },
        
- For sure, we need to add another route to our NATGW in order to achieve internet connectivity.

        "NATGatewayRoute" : {
                "Type" : "AWS::EC2::Route",
                "Properties" : {
                    "DestinationCidrBlock" : "0.0.0.0/0",
                    "NatGatewayId" : {"Ref" : "DMZ2NatGW"},
                    "RouteTableId" : {"Ref" : "PrivateRT"}
            }   
        },
        
- For security purposes, we are adding Network Access Control List **(NACL)** to our network in **subnet level** for public subnets as well as private subnets. Later on, we will add Security Group **(SG)** for the same purpose. However, that would be in **Elastic Compute Cloud (EC2) level**.

        "DMZPublicNACL" : {
            "Type" : "AWS::EC2::NetworkAcl",
            "Properties" : {
                "Tags" : [ {"Key" : "Name", "Value" : "DMZ-Public-Subnets-NACL"} ],
                "VpcId" : {"Ref" : "VPC"}
            }
        },
        "AppLayerNACL" : {
            "Type" : "AWS::EC2::NetworkAcl",
            "Properties" : {
                "Tags" : [ {"Key" : "Name", "Value" : "AppLayer-Private-Subnets-NACL"} ],
                "VpcId" : {"Ref" : "VPC"}
            }
        },
        
- Lets associate our public and private subnets with **NACLs**.

      "DMZ2NACLAssociation" : {
            "Type" : "AWS::EC2::SubnetNetworkAclAssociation",
            "DependsOn" : "DMZPublicNACL",
            "Properties" : {
                "NetworkAclId" : {"Ref" : "DMZPublicNACL"},
                "SubnetId" : {"Ref" : "DMZ2Public"}
            }
        },
        "AppLayer1NACLAssociation" : {
            "Type" : "AWS::EC2::SubnetNetworkAclAssociation",
            "DependsOn" : "AppLayerNACL",
            "Properties" : {
                "NetworkAclId" : {"Ref" : "AppLayerNACL"},
                "SubnetId" : {"Ref" : "AppLayer1Private"}
            }
        },
        
- It's time to add inbound **(Ingress)** and outbound **(Egress)** rules to our NACLs. Since NACLs are **stateless**, we have to add outbound rules whereas we do not need to add outbound rules to our SGs which are **statefull**.

        "DMZPublicSubnetsNACLIngress100" : {
            "Type" : "AWS::EC2::NetworkAclEntry",
            "DependsOn" : "DMZPublicNACL", 
            "Properties" : {
                "CidrBlock" : "0.0.0.0/0",
                "Egress" : false,
                "NetworkAclId" : {"Ref" : "DMZPublicNACL"},
                "PortRange" : {"From" : "22", "To" : "22"},
                "Protocol" : "6",
                "RuleAction" : "allow",
                "RuleNumber" : "100"
            }
        },
        "DMZPublicSubnetsNACLEgress100" : {
            "Type" : "AWS::EC2::NetworkAclEntry",
            "DependsOn" : "DMZPublicNACL", 
            "Properties" : {
                "CidrBlock" : "0.0.0.0/0",
                "Egress" : true,
                "NetworkAclId" : {"Ref" : "DMZPublicNACL"},
                "PortRange" : {"From" : "22", "To" : "22"},
                "Protocol" : "6",
                "RuleAction" : "allow",
                "RuleNumber" : "100"
            }
        },
        
        Adding Ingress and Egress rules on private subnet NACLs for our usecase.
        
        "AppLayerPrivateSubnetsNACLIngress100" : {
            "Type" : "AWS::EC2::NetworkAclEntry",
            "DependsOn" : "AppLayerNACL", 
            "Properties" : {
                "CidrBlock" : "10.0.0.0/16",
                "Egress" : false,
                "NetworkAclId" : {"Ref" : "AppLayerNACL"},
                "PortRange" : {"From" : "22", "To" : "22"},
                "Protocol" : "6",
                "RuleAction" : "allow",
                "RuleNumber" : "100"
            }
        },
        "AppLayerPrivateSubnetsNACLEgress100" : {
            "Type" : "AWS::EC2::NetworkAclEntry",
            "DependsOn" : "AppLayerNACL", 
            "Properties" : {
                "CidrBlock" : "0.0.0.0/0",
                "Egress" : true,
                "NetworkAclId" : {"Ref" : "AppLayerNACL"},
                "PortRange" : {"From" : "22", "To" : "22"},
                "Protocol" : "6",
                "RuleAction" : "allow",
                "RuleNumber" : "100"
            }
        },
        
- Now, we add Application Load Balancer **(ALB)** which will allow us to have internet connectivity as well as ditribute incoming traffic to healthy instances. 

         "ALB" : {
            "Type" : "AWS::ElasticLoadBalancingV2::LoadBalancer",
            "Properties" : {
                "IpAddressType" : "ipv4",
                "Name" : "AplicationLoadBalancer",
                "Scheme" : "internet-facing",
                "SecurityGroups" : [ {"Ref" : "ALBSG"} ],
                "Subnets" : [ {"Ref" : "DMZ1Public"}, {"Ref" : "DMZ2Public"} ],
                "Type" : "application"
            }
        },
        
- Now, we add **Bastion Host** instance to our VPC that will provide private connectivity from public subnets to private subnets. It acts as a proxy. Because of exposure to potential attack, Bastion Host will minimize chances of penetration. We also called as Jump Box. 

        "BastionInstance": {
            "Type": "AWS::EC2::Instance",
            "Properties": {
                "ImageId": "ami-0dd0ccab7e2801812",
                "InstanceType": "t2.micro",
                "KeyName": {"Ref": "WebServerKey"},
                "NetworkInterfaces": [
                    {"GroupSet": [{"Ref": "BastionHostSG"}],
                    "AssociatePublicIpAddress": "true",
                    "DeviceIndex": "0",
                    "DeleteOnTermination": "true",
                    "SubnetId": {"Ref" : "DMZ1Public"}}
                ],
                "Tags": [ {"Key": "Name", "Value": "bastion-host"} ]
            }
        },


- We are adding our Security Groups **(SGs)** for extra security layer. We consider SGs as firewalls for our local PCs. We add security group for ALB, Bastion Host, and private subnets (AppServerSG). SGs are statefull so we do not need to add outbound rules. 

        "ALBSG" : {
            "Type" : "AWS::EC2::SecurityGroup",
            "Properties" : {
                "GroupName" : "LoadBalancer-SG",
                "GroupDescription" : "WordPress-ALB",
                "VpcId" : {"Ref" :"VPC"},
                "SecurityGroupIngress" : [ 
                    {
                        "CidrIp" : "0.0.0.0/0",
                        "FromPort" : 80,
                        "IpProtocol" : "tcp",
                        "ToPort" : 80
                    },
                    {
                        "CidrIp" : "0.0.0.0/0",
                        "FromPort" : 443,
                        "IpProtocol" : "tcp",
                        "ToPort" : 443
                    }      
                ],
                "Tags" : [{"Key" : "Name", "Value" : "LoadBalancer-SG"}] 
            }
        },
        "BastionHostSG": {
            "Type" : "AWS::EC2::SecurityGroup",
            "Properties" : {
                "GroupName" : "BashtionHost-SG",
                "GroupDescription" : "WordPress-ALB",
                "VpcId" : {"Ref" :"VPC"},
                "SecurityGroupIngress" : [ 
                    {
                        "CidrIp" : "0.0.0.0/0",
                        "FromPort" : 22,
                        "IpProtocol" : "tcp",
                        "ToPort" : 22
                    }
                ],
                "Tags" : [{"Key" : "Name", "Value" : "BashtionHost-SG"}]
                
            }
        },
        "AppServerSG": {
            "Type" : "AWS::EC2::SecurityGroup",
            "Properties" : {
                "GroupName" : "AppLayer-SG",
                "GroupDescription" : "WordPress-ALB",
                "VpcId" : {"Ref" :"VPC"},
                "SecurityGroupIngress" : [ 
                    {
                        "SourceSecurityGroupId" : {"Ref" : "BastionHostSG"},
                        "FromPort" : 22,
                        "IpProtocol" : "tcp",
                        "ToPort" : 22
                    },
                    {
                        "SourceSecurityGroupId" : {"Ref":"ALBSG"},
                        "FromPort" : 80,
                        "IpProtocol" : "tcp",
                        "ToPort" : 80
                    }
                ],
                "Tags" : [{"Key" : "Name", "Value" : "AppLayer-SG"}] 
            }
        },
       
- Lets add components of our ALB which are **Listener** and **Target Groups** that will allow us to register our instances. 

        "Listener": {
            "Type": "AWS::ElasticLoadBalancingV2::Listener",
            "Properties": {
                "DefaultActions": [
                    {
                        "Type": "forward",
                        "TargetGroupArn": {"Ref" : "TargeGroup"}
                    }
                ],
                "LoadBalancerArn": {"Ref": "ALB"},
                "Port": 80,
                "Protocol": "HTTP"
            }
        },
        "TargetGroup" : {
            "Type" : "AWS::ElasticLoadBalancingV2::TargetGroup",
            "Properties" : {
                "HealthCheckEnabled" : true,
                "HealthCheckIntervalSeconds" : 10,
                "HealthCheckPath" : "/",
                "HealthCheckPort" : 80,
                "HealthCheckProtocol" : "HTTP",
                "HealthyThresholdCount" : 2,
                "Name" : "TargetGroup",
                "Port" : 80,
                "Protocol" : "HTTP",
                "Tags" : [ {"Key" : "Name", "Value" : "Instance-Target-Group"} ],
                "TargetType" : "instance",
                "UnhealthyThresholdCount" : 2,
                "VpcId" : {"Ref" : "VPC"}
            }
        },
        
- We need to add **Launch Configuration** to configure our application which will include **user data**. 

        "LaunchConfiguration" : {
            "Type" : "AWS::AutoScaling::LaunchConfiguration",
            "Properties" : {
                "AssociatePublicIpAddress" : false,
                "InstanceMonitoring": "true",
                "ImageId" : "ami-0dd0ccab7e2801812",
                "InstanceType" : "t2.micro",
                "KeyName" : {"Ref" : "WebServerKey"},
                "PlacementTenancy" : "default",
                "SecurityGroups" : [ {"Ref" : "AppServerSG"} ],
                "UserData" : {
                    "Fn::Base64" : {
                        "Fn::Join" : [
                            "",
                            [
                            "#!/bin/bash\n",
                            "yum update -y\n",
                            "wget http://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm\n",
                            "yum localinstall -y mysql57-community-release-el7-8.noarch.rpm\n",
                            "yum install -y mysql-community-server\n",
                            "yum install -y httpd php php-mysqlnd git\n",
                            "cd /var/www/html\n",
                            "wget https://wordpress.org/wordpress-5.1.1.tar.gz\n",
                            "tar -xzf wordpress-5.1.1.tar.gz\n",
                            "cp wordpress/wp-config-sample.php wp-config.php\n",
                            "echo '<?php phpinfo(); ?>' > /var/www/html/phpinfo.php\n",
                            "echo 'Hello World' > /var/www/html/index.html\n",
                            "groupadd www\n",
                            "usermod -aG www ec2-user\n",
                            "chown -R root:www /var/www\n",
                            "service httpd start\n",
                            "chkconfig httpd on\n",
                            "service mysqld start\n",
                            "chkconfig mysqld on\n",
                            "service httpd restart\n",
                            "service mysqld restart"
                            ]
                        ]
                    }
                }
            }
        },
        
- Lets add our **Auto Scaling Group (ASG)** that will allow us to increase and decrease number of instances based on traffic. 

        "AutoScalingGroup" : {
            "Type" : "AWS::AutoScaling::AutoScalingGroup",
            "DependsOn" : "ALB",
            "Properties" : {
                "AutoScalingGroupName" : "AppLayer-Auto-Scaling-Group",
                "Cooldown" : "300",
                "DesiredCapacity" : "3",
                "HealthCheckType" : "ELB",
                "HealthCheckGracePeriod" : 300,
                "LaunchConfigurationName" : {"Ref" : "LaunchConfiguration"},
                "MaxSize" : "5",
                "MinSize" : "2",
                "Tags" : [ {"PropagateAtLaunch": true, "Key" : "Name", "Value" : "Instance-WordPress"} ],
                "TargetGroupARNs" : [ {"Ref" : "TargeGroup"} ],
                "VPCZoneIdentifier" : [ {"Ref" : "AppLayer1Private"}, {"Ref" : "AppLayer2Private"} ]
            }
        },
        
- Lets add **policy** for our ASG. At the end of the day we need to create a policy to tell ASG how to act if load is increased or decreased and based on what we need to take an action. Therefore, we add a policy for that reason.

        "ASGCPUPolicy" : {
            "Type" : "AWS::AutoScaling::ScalingPolicy",
            "Properties" : {
                "AdjustmentType" : " PercentChangeInCapacity",
                "AutoScalingGroupName" : {"Ref" : "AutoScalingGroup"},
                "PolicyType" : "TargetTrackingScaling",
                "TargetTrackingConfiguration" : {
                    "TargetValue": 40.0,
                    "PredefinedMetricSpecification" : {
                        "PredefinedMetricType" : "ASGAverageCPUUtilization"
                    }
                }
            }
         }  
         
###### STEPS TO CONFIGURE GITHUB AGENT AND CI/CD 

1.  Go to Github and click on **_Actions_** tab then click on **_New Workflow_**
2.  Click on **_"Set up this workflow"_** and give a name for your yaml file..
3.  Paste your yaml script inside .yml file so each time there is a push on master branch and pull request is done then run this code again and CI/CD pipeline gets     triggerred automatically.
4.  Click on **_"Start Commit"_** and put your commit message and commit changes. 

Here, we are deploying everything in CI/CD pipeline so all deployment process continously get triggered. 

###### Continious Integration - Continous Development - Continous Deployment

Our main goal is to find the bugs as early as possible!
    
5. Then add your **_Access_Key_ID_** and **_Secret_Key_ID_**, so go to **Settings** and click on **Secrets**
6. Then click on **New Repository Secret**. Name your Access Key ID then paste your Accces Key ID which is provided when user role is created in **CSV** file.
7. Click on **Add Secret** then your Access Key ID will be stored.
8. Then do the same steps to store **Secret Key ID**.
9. Now, lets go ahead and add our agent.
10. Add Ubuntu instance to your infrastructure and spin up our machine. 
11. Wait for your instance to be initialized completely.
12. Once our instance is initialized we can go to terminal and ssh to our instance.
13. Meanwhile, go to Github and continue configuring your agent.
14. Go to **Settings** under your repository and click on **Actions**
15. Click on **Runners** and **_new_self_hosted_runner_** top right corner.
16. Since we chose Ubuntu as our instance, choose **Linux** option in the middlle.
17. Check your instance on AWS Console, and follow all commands on Github page if your instance is up and running. 
18. In order to shh to your instance, take **public ip adress** of your instance and go to terminal.
19. Go to terminal and **ssh to your instance**:  ssh -i ~/Downloads/demo.pem ubuntu@publicIP
      - If you get bad permissions after doing ssh to your instance, you need to change mode to read only for the key
          and run the **command:** chmod 400 ~/Downloads/pemfilename.pem
      - Check the permissions with the **command:** ls -l ~/Downloads/pemfilename.pem
21. After you are able to go to inside your instance follow all the commands on Github to configure your agent.
22. If you did not install **aws cli** on your instance, you need to do that before running CI/CD pipeline.
23. Once you get **Settings** saved on your instance, your run "./run.sh" command to run your CI/CD.
24. Then if you make any dummy change to your json file which has your CFT then push it to remote repo, your job will start running and so CI/CD.

###### How To Check Whether Your Application Is Running?

         After successfully complete your CI/CD job, it is verifiable that your application is up and running online. 
      To do that, go to your Application Load Balancer (ALB) service then expand below section of the page where 
      description is placed. Afterwards, copy DNS name and paste it to your browser.
      
      If you see the same message on the screen as in your Launch Configuration resource in CFT json file. Then it 
      means your application is up and running!

        
        
  
  
  
  
  
  
  
  
  
  
  













