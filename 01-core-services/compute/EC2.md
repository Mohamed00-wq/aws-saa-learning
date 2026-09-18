# EC2 — Elastic Compute Cloud 

## Purpose

 * In AWS **ec2*(Elastic Compute Cloud)** is used to create and run virtual servers in cloud.

 * Think of EC2 as renting a computer from AWS instead of buying and maintaining a phisycal server.


## How it work

 * EC2 allows you to create and run virtual servers called **instances** in the AWS cloud. 
 
 * when you launchning an instance you chose an AMI that provides the OS 

 * an instance type that determines  the CPU and RAM  

 * configure networking  
 
 * configure SGs to controll network traffic 


Suppose you developed a Java web application on your laptop

Users → Internet → EC2 instance → Your Java application → Database

You can create an EC2 instance choose Ubuntu Linux install Java upload your application and start it Your EC2 instance can then serve the application over the internet.


## When to use EC2 

we use EC2 whene we need  control over a virtual server and its environment

     * we need  to run a custom application on a linux or Windows server

     * we need a specific CPU RAM storage or networking 


## When to not use 

Don't choose EC2  when AWS  a managed service  that already solves the problem and we don't need server level control


## Important features 

     * Instance Types chose CPU RAM network and other resources our workload needs 
     * AMI (amazon machine image) a template containing the OS and software config 
     * EBS (elastic block store) presistent block storage attached to EC2 instances 
     * instance store Temporary local storage physicaly attached to the host data is lost whene the instace stopped terminated
     * Security Groups virtual firawalls that control inbound and outbound  traffic for EC2
     * Elastic IP  a statuc public ipv4 address that can be associated with  an EC2 
     * Key pairs used  to securely connect to instances ssh for linux
     * auto scalling automatically adds or removes ec2 instances according to demand
     * Elasitc load balanciing distributes traffic across multiple ec2 instances 
     * Placement Groups control how ec2 instances are physically places to optimize for performance low latency or fault tolerance 
     * purchasing options on demand, reserved instances/ savings plans, spot instances and dedicated hosts/instances 
     * instance lifecycle ec2 instances can be pending → running → stopping/stopped → terminated
     * IAM Roles  give an ec2 instance  permession to access  aws services  without  storing aws access keys on the server 


## limitations 

EC2 is powerful but also has some important limitations 
     **you manage the server**
     **scaling is not automatic by default**
     **higher operational overhead** compare with another services ec2 need more addministration 
     **instance failure is possible**
     **instance limits**  number vCPU elastic ips, etc  
     **instance store is temporary**
     **costs continue while running**
     **some workloads require specific instance type**
     **some workloads require specific instance type**
     **regional/AZ dependency** an ec2 instance exits in a specific aws region / availability zone 


## trade-off

     trade-off means choosing between two or more technical solutions where each option has advantages and disadvantages you can't maximize everything at once so you optimize for what matters most in a given scenario.