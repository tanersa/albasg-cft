                      *** STEPS TO CREATE CLOUD FORMATION TEMPLATE (CFT) ***
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
        "DMZ2NACLAssociation" : {
            "Type" : "AWS::EC2::SubnetNetworkAclAssociation",
            "DependsOn" : "DMZPublicNACL",
            "Properties" : {
                "NetworkAclId" : {"Ref" : "DMZPublicNACL"},
                "SubnetId" : {"Ref" : "DMZ2Public"}
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

        
  
  
  
  
  
  
  
  
  
  
  













