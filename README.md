                      *** STEPS TO CREATE CLOUD FORMATION TEMPLATE (CFT) ***
  ![alt text](https://github.com/tanersa/albasg-cft/blob/master/diagram/CloudFormationConcept.png)
  
  Our main goal is to have immutable infrastructure, so we build our VPC in order to achieve network isolation. Therefore, the highest priority in Software Development Life Cycle is the SECURITY! Cloudformation will provison our infrastructure. Eventually, CFT will be used to create a stack. We usually use json/ymal format for creating CFT. 
  
  At the very beginning of CFT, we define our CFT name as a description by following naming convention. Then, we would have keyname with logical id as well as the type as below:
  
      "AWSTemplateFormatVersion": "2010-09-09",
    "Description": "This CFT provisions AWS VPC, ALB and ASG for Wordpress HA",
    "Parameters": 
        "WebServerKeyName": 
            "Description": "This key will be used to SSH to our EC2 instances",
            "Type": "AWS::EC2::KeyPair::KeyName"
       
    
  
  


