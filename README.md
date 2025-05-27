
# $${\color{orange} \textbf{Project}: \textbf{3-tier} \ \textbf{Student} \ \textbf{App}}$$
  # 🌟 3-Tier Student Application Deployment on AWS

Welcome to the **3-Tier Student App** project! This guide walks you through deploying a scalable, secure, and robust 3-tier web application on AWS using a VPC, EC2 instances, RDS, and more. Follow these steps to set up a student management application with an Nginx frontend, Tomcat application server, and MySQL database.

---

## 📋 Prerequisites

Before you begin, ensure you have the following:
- **AWS Account** with access to VPC, EC2, and RDS services
- Basic understanding of AWS networking (VPC, subnets, route tables)
- SSH client and key pair for accessing EC2 instances
- Familiarity with Linux commands

---

## 🏗️ Architecture Overview

This project sets up a **3-tier architecture**:
1. **Presentation Layer**: Nginx server in a public subnet
2. **Application Layer**: Tomcat server in a private subnet
3. **Data Layer**: MySQL database (RDS) in a private subnet

![Architecture Diagram](https://github.com/abhipraydhoble/Project-3-tier-Student-App/raw/main/assets/vpc-flow.png)

---

## 🛠️ Step-by-Step Setup

### 1. Create a VPC
Set up a Virtual Private Cloud (VPC) to host the application.

- **Name**: `VPC-3-tier`
- **CIDR Block**: `192.168.0.0/16`

### 2. Create Subnets
Create four subnets for different components of the application.

| Subnet Name               | Type       | CIDR Block       |
|---------------------------|------------|------------------|
| `Public-Subnet-Nginx`     | Public     | `192.168.1.0/24` |
| `Public-Subnet-LB`        | Public     | `192.168.4.0/24` |
| `Private-Subnet-Tomcat`   | Private    | `192.168.2.0/24` |
| `Private-Subnet-Database` | Private    | `192.168.3.0/24` |

### 3. Configure Internet Gateway
Enable internet access for public subnets.

- **Name**: `IGW-3-tier`
- **Action**: Attach to `VPC-3-tier`

### 4. Set Up NAT Gateway
Allow private subnets to access the internet for updates.

- **Name**: `NAT-3-tier`
- **Location**: Place in `Public-Subnet-Nginx`

### 5. Configure Route Tables
Create route tables to manage traffic.

- **Public Route Table (`RT-Public-Subnet`)**
  - Associate with: `Public-Subnet-Nginx`, `Public-Subnet-LB`
  - Route: `0.0.0.0/0` → `IGW-3-tier`
- **Private Route Table (`RT-Private-Subnet`)**
  - Associate with: `Private-Subnet-Tomcat`, `Private-Subnet-Database`
  - Route: `0.0.0.0/0` → `NAT-3-tier`

---

## 🚀 Deploy EC2 Instances

Create EC2 instances for each tier with the specified configurations.

| Instance Name             | Subnet                     | Ports Allowed |
|---------------------------|----------------------------|---------------|
| `Nginx-Server-Public`     | `Public-Subnet-Nginx`      | 80, 22        |
| `Tomcat-Server-Private`   | `Private-Subnet-Tomcat`    | 8080, 22      |
| `Database-Server-Private` | `Private-Subnet-Database`  | 3306, 22      |

![EC2 Instances](https://github.com/abhipraydhoble/Project-3-tier-Student-App/raw/main/assets/instances.png)

---

## 🗄️ Set Up RDS Database

Create a MySQL database using AWS RDS.

- **Creation Method**: Standard Create
- **Template**: Free Tier
- **DB Name**: `database-1`
- **Username**: `admin`
- **Password**: `Passwd123$`
- **VPC**: `VPC-3-tier`
- **Subnet**: `Private-Subnet-Database`
- **Public Access**: No
- **Availability Zone**: No preference
- **Security Group**: Add inbound rule for port `3306`

![RDS Setup](https://github.com/abhipraydhoble/Project-3-tier-Student-App/raw/main/assets/database.png)

---

## 🔗 Connect to Instances

### Connect to `Nginx-Server-Public`
1. SSH into the instance using your key pair.
2. Create a file for the private key:
   ```bash
   vim 3-tier-key.pem
   ```
3. Copy and paste the private key into the file.
4. Change the hostname for clarity:
   ```bash
   sudo hostnamectl set-hostname nginx-server
   ```

![Nginx Server](https://github.com/abhipraydhoble/Project-3-tier-Student-App/raw/main/assets/nginx-server.png)

### Connect to `Database-Server-Private`
1. SSH into the instance:
   ```bash
   ssh -i 3-tier-key.pem ec2-user@<database-server-private-ip>
   ```
2. Install MariaDB client:
   ```bash
   sudo -i
   yum install mariadb105-server -y
   systemctl start mariadb
   systemctl enable mariadb
   ```
3. Connect to the RDS database:
   ```bash
   mysql -h <rds-endpoint> -u admin -pPasswd123$
   ```
   Replace `<rds-endpoint>` with the actual RDS endpoint.

4. Create the `studentapp` database and table:
   ```sql
   show databases;
   create database studentapp;
   use studentapp;
   CREATE TABLE if not exists students (
       student_id INT NOT NULL AUTO_INCREMENT,
       student_name VARCHAR(100) NOT NULL,
       student_addr VARCHAR(100) NOT NULL,
       student_age VARCHAR(3) NOT NULL,
       student_qual VARCHAR(20) NOT NULL,
       student_percent VARCHAR(10) NOT NULL,
       student_year_passed VARCHAR(10) NOT NULL,
       PRIMARY KEY (student_id)
   );
   show tables;
   exit
   ```

![Database Login](https://github.com/abhipraydhoble/Project-3-tier-Student-App/raw/main/assets/login_into_database.png)

---

## ⚙️ Configure Tomcat Server

1. SSH into `Tomcat-Server-Private`:
   ```bash
   ssh -i 3-tier-key.pem ec2-user@<tomcat-server-private-ip>
   sudo -i
   ```
2. Install Java:
   ```bash
   yum install java -y
   ```
3. Download and extract Tomcat:
   ```bash
   curl -O https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.105/bin/apache-tomcat-9.0.105.tar.gz
   tar -xzvf apache-tomcat-9.0.105.tar.gz -C /opt/
   ```
4. Deploy the student application:
   ```bash
   cd /opt/apache-tomcat-9.0.105/webapps
   curl -O https://s3-us-west-2.amazonaws.com/studentapi-cit/student.war
   ```
5. Add MySQL connector:
   ```bash
   cd ../lib
   curl -O https://s3-us-west-2.amazonaws.com/studentapi-cit/mysql-connector.jar
   ```
6. Modify `context.xml`:
   ```bash
   cd ../conf
   vim context.xml
   ```
   Add the following at line 21:
   ```xml
   <Resource name="jdbc/TestDB" auth="Container" type="javax.sql.DataSource"
             maxTotal="100" maxIdle="30" maxWaitMillis="10000"
             username="admin" password="Passwd123$" driverClassName="com.mysql.jdbc.Driver"
             url="jdbc:mysql://<rds-endpoint>:3306/studentapp"/>
   ```
   Replace `<rds-endpoint>` with the actual RDS endpoint.

7. Start Tomcat:
   ```bash
   cd ../bin
   chmod +x catalina.sh
   ./catalina.sh start
   exit
   ```

![Tomcat Setup](https://github.com/abhipraydhoble/Project-3-tier-Student-App/raw/main/assets/tomcat-server.png)

---

## 🌐 Configure Nginx Server

1. Return to `Nginx-Server-Public`:
   ```bash
   ssh -i 3-tier-key.pem ec2-user@<nginx-server-public-ip>
   ```
2. Install and configure Nginx:
   ```bash
   sudo yum install nginx -y
   sudo vim /etc/nginx/nginx.conf
   ```
3. Add the following at line 47 (between `error` and `location` blocks):
   ```nginx
   location / {
       proxy_pass http://<tomcat-private-ip>:8080/student/;
   }
   ```
   Replace `<tomcat-private-ip>` with the private IP of the Tomcat instance.

4. Save and start Nginx:
   ```bash
   :wq
   sudo systemctl start nginx
   ```

![Nginx Config](https://github.com/user-attachments/assets/31feb10d-d005-4240-b629-839b8a596777)

---

## 🎉 Test the Application

1. Open a browser and navigate to the public IP of `Nginx-Server-Public`.
2. You should see the student application interface.
3. Register and manage student records through the web interface.

![Application Output](https://github.com/abhipraydhoble/Project-3-tier-Student-App/raw/main/assets/nginx-output.png)
![Register Students](https://github.com/abhipraydhoble/Project-3-tier-Student-App/raw/main/assets/register-students.png)

---

## 🔐 Security Best Practices

- Restrict security group rules to allow only necessary ports.
- Store the `3-tier-key.pem` securely and never expose it publicly.
- Use AWS Secrets Manager for sensitive data like RDS credentials.
- Regularly update and patch EC2 instances and software.

---

## 🚀 Next Steps

- **Add a Load Balancer**: Place it in `Public-Subnet-LB` for high availability.
- **Enable Auto Scaling**: Scale Tomcat instances based on demand.
- **Monitor with CloudWatch**: Set up alarms and logs for performance monitoring.
- **Backup RDS**: Configure automated backups for data safety.

---

## 📚 Resources

- [AWS VPC Documentation](https://docs.aws.amazon.com/vpc/)
- [AWS RDS Documentation](https://docs.aws.amazon.com/AmazonRDS/)
- [Tomcat 9 Documentation](https://tomcat.apache.org/tomcat-9.0-doc/)
- [Nginx Documentation](https://nginx.org/en/docs/)

Happy deploying! 🚀
