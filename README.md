# **CloudFormation Template Fix Documentation**

## **Problem Analysis & Fixes**
This document outlines the key issues that were preventing the webpage from loading through the Application Load Balancer (ALB) and how they were resolved.



### **1. ALB Deployed in Private Subnets (Problem 1)**
#### **Issue**  
The ALB was initially configured in **private subnets**, making it inaccessible from the internet. Since ALBs must be in **public subnets** to serve web traffic, users couldn't reach the webpage.

#### **Fix**  
- Changed the `Subnets` property in the `ApplicationLoadBalancer` resource to reference **public subnets** (`PublicSubnet` and `PublicSubnet2`).
- Ensured the subnets had `MapPublicIpOnLaunch: true` and were associated with an Internet Gateway (IGW).

#### **Before (Incorrect)**
```json
"Subnets": [
  { "Ref": "PrivateSubnet" },
  { "Ref": "PrivateSubnet2" }
]
```

#### **After (Correct)**
```json
"Subnets": [
  { "Ref": "PublicSubnet" },
  { "Ref": "PublicSubnet2" }
]
```



### **2. Target Group Not Associated with Auto Scaling Group (Problem 2)**
#### **Issue**  
The Auto Scaling Group (ASG) was launching instances, but they were **not registered** with the ALB’s target group, meaning the ALB had no healthy targets to route traffic to.

#### **Fix**  
- Added `TargetGroupARNs` to the `AutoScalingGroup` resource, linking it to the ALB’s target group.
- Ensured the `ALBTargetGroup` was properly configured with HTTP health checks.

#### **Before (Missing Association)**
```json
"AutoScalingGroup": {
  "Properties": {
    "VPCZoneIdentifier": [...],
    "LaunchTemplate": {...}
    // Missing TargetGroupARNs
  }
}
```

#### **After (Correct)**
```json
"AutoScalingGroup": {
  "Properties": {
    "VPCZoneIdentifier": [...],
    "LaunchTemplate": {...},
    "TargetGroupARNs": [
      { "Ref": "ALBTargetGroup" }
    ]
  }
}
```



### **3. Incorrect Health Check Path (Problem 3)**
#### **Issue**  
The ALB’s target group had a **typo** in the health check path (`/healthcheck.hmtl` instead of `/healthcheck.html`), causing health checks to fail and marking instances as unhealthy.

#### **Fix**  
- Corrected the `HealthCheckPath` in the `ALBTargetGroup` resource.

#### **Before (Incorrect)**
```json
"HealthCheckPath": "/healthcheck.hmtl"
```

#### **After (Correct)**
```json
"HealthCheckPath": "/healthcheck.html"
```


### **4. Security Group Misconfiguration (Problem 4)**
#### **Issue**  
- The **WebSG** (EC2 security group) only allowed **SSH (port 22)** from the ALB, but not **HTTP (port 80)**.
- The ALB security group was not properly allowing inbound HTTP traffic.

#### **Fix**  
1. **WebSG**: Modified to allow HTTP (port 80) from the ALB’s security group.
2. **ALB Security Group**: Ensured it allows **inbound HTTP (port 80)** from the internet (`0.0.0.0/0`).

#### **Before (Incorrect WebSG)**
```json
"WebSG": {
  "Properties": {
    "SecurityGroupIngress": [
      {
        "FromPort": 22,
        "ToPort": 22,
        "SourceSecurityGroupId": { "Ref": "LoadBalancerSecurityGroup" }
      }
      // Missing HTTP rule
    ]
  }
}
```

#### **After (Correct WebSG)**
```json
"WebSG": {
  "Properties": {
    "SecurityGroupIngress": [
      {
        "FromPort": 80,
        "ToPort": 80,
        "SourceSecurityGroupId": { "Ref": "LoadBalancerSecurityGroup" }
      },
      {
        "FromPort": 22,
        "ToPort": 22,
        "CidrIp": "0.0.0.0/0"
      }
    ]
  }
}
```



## **Deployment Steps**
### **1. Creating the Stack**
Use the AWS CLI or AWS Console to deploy the CloudFormation template.

#### **AWS CLI Command**
```bash
aws cloudformation create-stack \
  --stack-name owoseniStack \
  --template-body file://owoseni_stack.yml \
  --parameters ParameterKey=InstanceCount,ParameterValue=2 \
  --capabilities CAPABILITY_IAM
```
![image](https://github.com/user-attachments/assets/69b33fc9-3c5b-4461-ae8e-1d74d529940c)

### **2. Verify stack Deployment**
Run this to see if the stack deployed successfully:

```bash
aws cloudformation describe-stacks --stack-name Owoseni-stack --query "Stacks[0].StackStatus"
```

Expected final status: ```CREATE_COMPLETE```

![image](https://github.com/user-attachments/assets/a9d3e8d1-dcce-494a-bbbd-c2292ba2ee95)


### **3. Getting Stack Details**
After deployment, check the stack status and outputs.

#### **AWS CLI Command**
```bash
aws cloudformation describe-stacks --stack-name owoseniStack
```

#### **Expected Output**
```json
{
  "Stacks": [
    {
      "StackName": "WebAppStack",
      "Outputs": [
        {
          "OutputKey": "ALBDNSName",
          "OutputValue": "webapp-alb-1234567890.us-east-1.elb.amazonaws.com"
        }
      ]
    }
  ]
}
```

![image](https://github.com/user-attachments/assets/a9d901c8-676c-4240-936c-7512bb54ae15)


### **4. Accessing the Webpage via ALB**
Once the stack is in `CREATE_COMPLETE` status, the webpage can be accessed via the ALB’s DNS name.

#### **URL Format**
```
[http://<ALB-DNS-Name>](http://owosen-appli-fwyjx9mbfae4-748553299.us-east-1.elb.amazonaws.com/)
```



## **Verification Steps**
   
1. **Test Web Access**  
   - Open the ALB URL in a browser.
   - If it loads, the fixes were successful.

![image](https://github.com/user-attachments/assets/c2781181-12e8-4cf8-b1b3-f57304ad1277)



## **Conclusion**
By fixing the **subnet placement, target group association, health check path, and security group rules**, the ALB can now properly route traffic to the instances, allowing the webpage to load successfully.  

✅ **ALB in public subnets** → Internet-accessible  
✅ **ASG linked to Target Group** → Traffic routed to instances  
✅ **Correct health check path** → Instances marked healthy  
✅ **Proper security groups** → HTTP traffic allowed  

The webpage should now be accessible via the ALB’s DNS name. 🚀
