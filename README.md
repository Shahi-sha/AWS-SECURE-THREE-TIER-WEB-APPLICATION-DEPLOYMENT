# AWS-SECURE-THREE-TIER-WEB-APPLICATION-DEPLOYMENT
# AWS Three-Tier Web Application Architecture

## 1. Project Overview

This project demonstrates how to design and deploy a three-tier web application on Amazon Web Services (AWS). The application uses AWS networking, an internet-facing Application Load Balancer, a private EC2 instance running Docker containers, and Amazon RDS for MySQL database storage.

The project also incorporates AWS Identity and Access Management (IAM), AWS Systems Manager Session Manager, Amazon CloudWatch, and Amazon Simple Notification Service (SNS) for secure administration, logging, monitoring, and notifications.

The objective is to understand how different AWS services work together to build a secure, maintainable, and monitored cloud application.

## 2. Project Objectives

- Design an AWS Virtual Private Cloud (VPC).
- Create public and private subnets across two Availability Zones.
- Configure Internet Gateway, NAT Gateway, and route tables.
- Deploy a private EC2 application server.
- Install Docker and Docker Compose on Amazon Linux 2023.
- Run an Nginx reverse proxy and Flask backend in containers.
- Connect the Flask application to Amazon RDS for MySQL.
- Configure security groups to control network traffic.
- Use IAM roles and Systems Manager Session Manager for administrative access.
- Configure CloudWatch logging and alarms.
- Integrate SNS for email notifications.

## 3. Technologies and AWS Services Used

| Technology | Purpose |
|---|---|
| Amazon VPC | Isolated virtual network |
| Public and private subnets | Network segmentation |
| Internet Gateway | Internet connectivity for public resources |
| NAT Gateway | Outbound internet access for private resources |
| Application Load Balancer | Distributes incoming HTTP requests |
| Amazon EC2 | Hosts the application containers |
| Amazon Linux 2023 | Operating system for the EC2 instance |
| Docker | Runs application components in containers |
| Docker Compose | Defines and manages multiple containers |
| Nginx | Reverse proxy and HTTP entry point on the EC2 host |
| Python and Flask | Backend application |
| Amazon RDS for MySQL | Managed relational database |
| IAM | Identity and permissions management |
| AWS Systems Manager | Secure instance administration |
| Amazon CloudWatch | Logs, metrics, and alarms |
| Amazon SNS | Email notifications |

## 4. Architecture Overview

The application follows a three-tier logical architecture.

### Tier 1: Presentation Tier

The presentation tier receives HTTP requests from users. The internet-facing Application Load Balancer accepts incoming traffic and forwards it to the Nginx reverse proxy running on the private EC2 instance.

### Tier 2: Application Tier

The application tier consists of an EC2 instance running Docker containers.

- Nginx listens on port 80.
- Nginx forwards requests to the Flask backend on port 8000.
- Flask processes application requests and communicates with MySQL.

### Tier 3: Database Tier

Amazon RDS for MySQL stores the application's data. The database is configured without public access and accepts database connections on port 3306 only from the application security group.

### Supporting Services

IAM, Systems Manager, CloudWatch, and SNS provide administrative access, permissions, observability, and alerting.

**Request flow:**

```text
User Browser
     |
     | HTTP :80
     v
Internet Gateway
     |
     v
Application Load Balancer
     |
     | HTTP :80
     v
Private EC2 Instance
     |
     v
Nginx Docker Container
     |
     | HTTP :8000
     v
Flask Docker Container
     |
     | MySQL :3306
     v
Amazon RDS for MySQL
```

The ALB is publicly accessible, while the EC2 instance and RDS database are not directly accessible from the public internet.

## 5. AWS VPC and Network Configuration

### 5.1 Create the VPC

Create a VPC named `three-tier-vpc`.

Configuration:

- IPv4 CIDR: `10.0.0.0/16`
- Tenancy: Default
- DNS resolution: Enabled
- DNS hostnames: Enabled

**Explanation**

A VPC provides an isolated network for the application's AWS resources.

The CIDR block `10.0.0.0/16` defines the private IPv4 address space available for the subnets.

Enabling DNS resolution and DNS hostnames supports DNS-based communication between resources.

### 5.2 Create Six Subnets

Create three subnet types across two Availability Zones in the Mumbai region.

| Subnet | Availability Zone | CIDR |
|---|---|---|
| public-a | ap-south-1a | 10.0.1.0/24 |
| public-b | ap-south-1b | 10.0.2.0/24 |
| app-a | ap-south-1a | 10.0.11.0/24 |
| app-b | ap-south-1b | 10.0.12.0/24 |
| db-a | ap-south-1a | 10.0.21.0/24 |
| db-b | ap-south-1b | 10.0.22.0/24 |

Each `/24` subnet contains 256 IPv4 addresses, of which AWS reserves five.

**Explanation**

- Public subnets host resources that require internet-facing connectivity, such as the ALB and NAT Gateway.
- Application subnets host EC2 application resources.
- Database subnets provide network isolation for database resources.

Using two Availability Zones provides a foundation for improving availability and resilience.

**Important:** The database subnet CIDRs must be different. Use `10.0.22.0/24` for `db-b`, not a duplicate of `db-a`.

### 5.3 Create and Attach the Internet Gateway

Create an Internet Gateway named `three-tier-igw` and attach it to `three-tier-vpc`.

**Explanation**

An Internet Gateway connects a VPC to the internet when the relevant subnet route table, security groups, and network ACLs permit the traffic.

The Internet Gateway alone does not make every resource publicly accessible.

### 5.4 Create the NAT Gateway

Create a NAT Gateway in a public subnet and allocate an Elastic IP address to it.

**Explanation**

The NAT Gateway allows instances in private subnets to initiate outbound IPv4 connections to the internet without accepting unsolicited inbound internet connections through the NAT Gateway.

Typical use cases include downloading operating system updates and application packages.

The NAT Gateway must be in a public subnet with a route to the Internet Gateway.

For production high availability, consider a NAT Gateway in each Availability Zone to reduce cross-AZ dependency.

### 5.5 Configure Route Tables

Create and associate the appropriate route tables with the subnets.

**Public route table**

```text
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         Internet Gateway
```

Associate this route table with `public-a` and `public-b`.

**Private application route table**

```text
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         NAT Gateway
```

Associate it with the application subnets that require outbound internet access.

**Private database route table**

```text
Destination       Target
10.0.0.0/16       local
```

Associate it with `db-a` and `db-b`.

**Explanation**

Route tables determine where network traffic is directed.

The local route allows communication between resources in the VPC, subject to security controls. The public route table directs internet-bound traffic through the Internet Gateway, while the application route table directs outbound internet traffic through the NAT Gateway.

The database tier does not need a default internet route for basic connectivity with the application tier.

## 6. Configure Security Groups

Security groups act as stateful virtual firewalls attached to AWS resources.

Create three security groups in the VPC.

### 6.1 ALB Security Group: `alb-sg`

Inbound rule:

| Type | Port | Source |
|---|---:|---|
| HTTP | 80 | 0.0.0.0/0 |

**Explanation**

This rule allows users to access the public HTTP endpoint of the Application Load Balancer.

For production applications, configure HTTPS on port 443 using an appropriate TLS certificate.

### 6.2 Application Security Group: `app-sg`

Inbound rule:

| Type | Port | Source |
|---|---:|---|
| Custom TCP | 80 | `alb-sg` |

**Explanation**

Only traffic originating from the ALB security group is allowed to reach port 80 on the EC2 instance.

The instance does not require a public IP address for this architecture.

For SSM administration, avoid opening SSH port 22 to the internet.

### 6.3 Database Security Group: `db-sg`

Inbound rule:

| Type | Port | Source |
|---|---:|---|
| MySQL/Aurora | 3306 | `app-sg` |

**Explanation**

The rule permits MySQL traffic from resources associated with the application security group.

The database does not accept direct public internet connections. This reduces the database's exposure to external threats.

## 7. Create the Amazon RDS MySQL Database

Create an Amazon RDS database using the MySQL engine.

Example configuration:

- Database identifier: `three-tier-mysql`
- Engine: MySQL
- VPC: `three-tier-vpc`
- DB subnet group: `three-tier-db-subnet-group`
- Public access: No
- Port: 3306
- Security group: `db-sg`
- Deployment: Single-AZ for the learning project

Create the DB subnet group using `db-a` and `db-b`.

Use a secure master password, store it outside the repository, and retrieve it through an approved secrets-management mechanism.

**Explanation**

Amazon RDS provides a managed database service. AWS handles underlying database infrastructure tasks, while the application connects to the database using its endpoint, port, database name, username, and password.

The subnet group identifies the subnets available to the database.

Although the subnet group spans two Availability Zones, a Single-AZ database deployment does not provide the same availability as a Multi-AZ deployment.

After creating the database, copy its endpoint from the RDS console.

The endpoint is used by the Flask application to establish a connection to MySQL.

## 8. Create an IAM Role for EC2

Create an IAM role named `three-tier-ec2-role`.

Configuration:

- Trusted entity: AWS service
- Use case: EC2
- Managed policy: `AmazonSSMManagedInstanceCore`

Attach this role to the EC2 instance through its instance profile.

**Explanation**

IAM controls what AWS actions an identity can perform.

An EC2 instance profile makes the role's temporary credentials available to applications and AWS agents running on the instance.

The `AmazonSSMManagedInstanceCore` policy provides permissions required for Systems Manager core managed-instance operations.

The role can also be given narrowly scoped permissions to publish application and Nginx logs to CloudWatch.

Do not store long-lived AWS access keys directly on the EC2 instance or in the GitHub repository.

## 9. Launch the Private EC2 Instance

Create an EC2 instance with the following example settings:

- Name: `three-tier-app-server`
- Operating system: Amazon Linux 2023
- VPC: `three-tier-vpc`
- Subnet: `app-a`
- Auto-assign public IP: Disabled
- Security group: `app-sg`
- IAM instance profile: `three-tier-ec2-role`

**Explanation**

The EC2 instance hosts the Docker-based application stack.

Placing it in a private subnet reduces direct internet exposure.

The instance can be managed through Systems Manager Session Manager if the SSM agent is running, the IAM role is attached, and network connectivity to the Systems Manager endpoints is available.

## 10. Connect Using Systems Manager Session Manager

Open AWS Systems Manager, select Session Manager, choose the managed EC2 instance, and start a session.

The instance needs outbound connectivity to the Systems Manager service through suitable VPC endpoints or another permitted network path.

**Explanation**

Session Manager provides browser-based or CLI-based shell access without requiring an inbound SSH connection.

This helps reduce the need to expose port 22 or distribute SSH private keys to administrators.

## 11. Install Docker and Git

Run the following commands in the EC2 terminal:

```bash
sudo dnf update -y
sudo dnf install -y docker git
sudo systemctl enable --now docker
docker --version
git --version
```

### Command-by-command explanation

**`sudo dnf update -y`**

Updates installed packages using the DNF package manager.

**`sudo dnf install -y docker git`**

Installs Docker and Git.

**`sudo systemctl enable --now docker`**

Enables the Docker service to start automatically at boot and starts it immediately.

**`docker --version`**

Displays the installed Docker version.

**`git --version`**

Confirms that Git is installed.

Install Docker Compose using the official installation instructions appropriate for the EC2 operating system and CPU architecture. Verify the binary before executing it and check its version.

## 12. Clone the Application Repository

The project uses the Nginx-Flask-MySQL example from the Docker Awesome Compose repository.

```bash
git clone --depth 1 https://github.com/docker/awesome-compose.git
cd awesome-compose/nginx-flask-mysql
ls -la
```

### Explanation

**`git clone --depth 1 ...`**

Downloads the latest commit available on the repository's default branch without fetching the full Git history.

**`cd awesome-compose/nginx-flask-mysql`**

Changes the working directory to the example application.

**`ls -la`**

Lists files, including hidden configuration files.

Review the repository files before modifying them. The project adapts the example to use Amazon RDS rather than a locally hosted MySQL container.

## 13. Configure the Flask Backend

The Flask backend handles application requests and retrieves data from MySQL.

The main application file is `backend/hello.py`.

The project uses Python, Flask, and a MySQL connector.

### 13.1 Import Dependencies

```python
import os
from flask import Flask
import mysql.connector
```

**Explanation**

- `os` reads environment variables.
- `Flask` creates the web application.
- `mysql.connector` provides a Python interface for MySQL.

### 13.2 Create the Database Manager

```python
class DBManager:
    def __init__(self):
        self.connection = mysql.connector.connect(
            host=os.environ["DB_HOST"],
            port=int(os.getenv("DB_PORT", "3306")),
            user=os.environ["DB_USER"],
            password=os.environ["DB_PASSWORD"],
            database=os.environ["DB_NAME"]
        )
        self.cursor = self.connection.cursor()
```

**Explanation**

`class DBManager` groups the application's database operations.

`__init__` runs when an instance of the class is created.

`mysql.connector.connect()` opens a database connection using the supplied configuration.

- `DB_HOST`: RDS endpoint.
- `DB_PORT`: MySQL port, normally 3306.
- `DB_USER`: Database username.
- `DB_PASSWORD`: Database password.
- `DB_NAME`: Database name.

The connection cursor executes SQL statements.

These values should be supplied through environment variables or a suitable secrets-management service, not hard-coded into the source code.

### 13.3 Create the Blog Table

```python
def populate_db(self):
    self.cursor.execute(
        'CREATE TABLE IF NOT EXISTS blog '
        '(id INT AUTO_INCREMENT PRIMARY KEY, title VARCHAR(255))'
    )
```

**Explanation**

`populate_db()` creates the `blog` table if it does not already exist.

The table contains:

- `id`: An automatically incrementing integer and primary key.
- `title`: A text field that stores a blog title.

`IF NOT EXISTS` prevents an error when the table already exists.

### 13.4 Check Whether Data Exists

```python
self.cursor.execute('SELECT COUNT(*) FROM blog')
count = self.cursor.fetchone()[0]
```

**Explanation**

The SQL query counts the number of records in the table.

`fetchone()` returns the first result row, and `[0]` extracts its first column.

The application uses this count to decide whether initial sample data should be inserted.

### 13.5 Insert Sample Data

```python
if count == 0:
    self.cursor.executemany(
        'INSERT INTO blog (title) VALUES (%s)',
        [(f'Blog post #{i}',) for i in range(1, 5)]
    )
    self.connection.commit()
```

**Explanation**

The code inserts four sample blog posts when the table is empty.

`executemany()` executes the parameterized insert statement for multiple records.

The `%s` placeholder lets the database connector bind values safely instead of constructing SQL from string concatenation.

`commit()` saves the inserted records.

### 13.6 Retrieve Blog Titles

```python
def query_titles(self):
    self.cursor.execute('SELECT title FROM blog')
    return [row[0] for row in self.cursor.fetchall()]
```

**Explanation**

The SQL statement selects all blog titles.

`fetchall()` returns the result rows, and the list comprehension extracts the title from each row.

The function returns a list of titles to the Flask route.

### 13.7 Create the Flask Application

```python
server = Flask(__name__)
conn = None
```

**Explanation**

`Flask(__name__)` initializes the Flask application.

The `conn` variable initially holds `None`. The application uses it to initialize its database manager when the route is first requested.

### 13.8 Define the Homepage

```python
@server.route('/')
def listBlog():
    global conn

    if conn is None:
        conn = DBManager()
        conn.populate_db()

    rec = conn.query_titles()
    return ''.join(f'<div>Hello {title}</div>' for title in rec)
```

**Explanation**

`@server.route('/')` maps the root URL to the `listBlog()` function.

When a browser requests the homepage:

1. Flask checks whether the database manager has been initialized.
2. If not, it connects to MySQL.
3. It creates the table and inserts sample data if necessary.
4. It retrieves the blog titles.
5. It returns an HTML response containing the titles.

This is a demonstration implementation. A production application should use proper connection pooling, error handling, HTML escaping or templates, and a production-grade WSGI server.

### 13.9 Start Flask

```python
if __name__ == '__main__':
    server.run(host='0.0.0.0', port=8000)
```

**Explanation**

This starts the Flask development server when the file is executed directly.

`0.0.0.0` allows the server to listen on the container's network interfaces.

Port `8000` is the internal port used by Nginx to communicate with Flask.

The Flask port is not directly published to the internet.

## 14. Configure the Nginx Reverse Proxy

Nginx acts as the HTTP entry point inside the EC2 application environment.

The configuration forwards requests to the Flask backend using Docker's internal service discovery.

Example Nginx configuration:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Line-by-line explanation

**`server {`**

Begins an Nginx server configuration block.

**`listen 80;`**

Tells Nginx to listen for HTTP requests on port 80.

**`server_name _;`**

Defines a catch-all server name for this example configuration.

**`location / {`**

Matches requests for the root path and other paths beginning with `/`.

**`proxy_pass http://backend:8000;`**

Forwards requests to the Docker Compose service named `backend` on port 8000.

The service name resolves through the Docker network.

**`proxy_set_header Host $host;`**

Passes the original host header to the backend.

**`proxy_set_header X-Real-IP $remote_addr;`**

Passes the address of the direct client connected to Nginx.

**`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`**

Maintains the forwarded-for header containing client and proxy addresses.

**`proxy_set_header X-Forwarded-Proto $scheme;`**

Passes the protocol used by the client connection to Nginx.

**`}`**

Closes the configuration blocks.

In a production environment, ensure the application trusts forwarded headers only from known proxies and configures the client IP chain correctly.

## 15. Configure Docker Compose

Docker Compose defines how the backend and Nginx containers are built, connected, and started.

Example configuration:

```yaml
services:
  backend:
    build:
      context: backend
      target: builder
    restart: always
    env_file:
      - .env
    expose:
      - "8000"
    networks:
      - frontnet

  proxy:
    build: proxy
    restart: always
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - frontnet

networks:
  frontnet:
```

### Line-by-line explanation

**`services:`**

Defines the containers managed by Docker Compose.

**`backend:`**

Defines the Flask backend service.

**`build:`**

Tells Docker Compose to build the backend image locally.

**`context: backend`**

Uses the `backend` directory as the Docker build context.

**`target: builder`**

Selects the `builder` stage from the backend Dockerfile.

**`restart: always`**

Configures Docker to restart the container automatically under the applicable restart conditions.

**`env_file:`**

Loads environment variables from a file.

**`- .env`**

Specifies the environment file containing the database connection configuration.

**`expose: - "8000"`**

Documents the backend's internal listening port and makes it available to connected Compose services. It does not publish the port directly on the EC2 host.

**`networks: - frontnet`**

Connects the backend to the Compose network.

**`proxy:`**

Defines the Nginx reverse proxy service.

**`build: proxy`**

Uses the `proxy` directory as the Nginx image build context.

**`ports: - "80:80"`**

Maps host port 80 to container port 80, allowing the ALB to reach Nginx through the EC2 instance.

**`depends_on: - backend`**

Sets a startup dependency so Compose starts the backend before the proxy. This alone does not guarantee that Flask is ready to accept requests.

**`networks: - frontnet`**

Connects Nginx to the same Docker network as Flask.

**`networks: frontnet:`**

Declares the shared network used by the containers.

Docker Compose provides DNS-based service discovery, allowing Nginx to resolve `backend` without using a fixed container IP address.

## 16. Configure the Database Environment

Create a local `.env` file containing the database connection parameters.

Use this template:

```dotenv
DB_HOST=your-rds-endpoint
DB_PORT=3306
DB_NAME=example
DB_USER=your-database-user
DB_PASSWORD=your-secure-password
```

Replace the placeholder values with the correct values for your environment.

**Explanation**

- `DB_HOST` identifies the RDS instance.
- `DB_PORT` specifies the MySQL service port.
- `DB_NAME` specifies the database that Flask will use.
- `DB_USER` identifies the database account.
- `DB_PASSWORD` provides the account password.

The `example` database must exist before the application attempts to use it.

For a real deployment, prefer AWS Secrets Manager or another approved secret-injection mechanism rather than relying on a plain-text environment file.

Add `.env` to `.gitignore`:

```gitignore
.env
*.pem
*.key
__pycache__/
*.pyc
```

Do not commit real database passwords, private keys, AWS access keys, or account-specific secret configuration to GitHub.

## 17. Create the Application Database

Connect to MySQL using an authorized administrative session or a MySQL client from the private application environment.

Example SQL:

```sql
CREATE DATABASE example;
SHOW DATABASES;
```

**Explanation**

`CREATE DATABASE example;` creates the database.

`SHOW DATABASES;` lists databases available to the connected user.

Use a dedicated application database user with only the privileges the application requires. Avoid granting unnecessary administrative privileges.

Update `DB_NAME` in the application's local environment configuration to `example` and restart the application containers if needed.

## 18. Build and Start the Containers

From the application directory, run:

```bash
docker-compose config --quiet
docker-compose up -d --build
docker-compose ps
docker-compose logs --tail=100
```

If using the Compose plugin, the equivalent command prefix is `docker compose`.

### Explanation

**`docker-compose config --quiet`**

Validates the Compose configuration without printing the full configuration.

**`docker-compose up -d --build`**

Builds the images, creates or updates the containers, and starts the services in the background.

**`docker-compose ps`**

Displays the current state of the Compose containers.

**`docker-compose logs --tail=100`**

Displays recent logs for the services.

Check the output for database connection errors, missing environment variables, image build failures, and Nginx configuration errors.

The expected application flow is:

```text
Browser
   |
   v
Application Load Balancer
   |
   v
Nginx :80
   |
   v
Flask :8000
   |
   v
Amazon RDS MySQL :3306
```

## 19. Configure the Target Group

Create an EC2 instance target group named `three-tier-tg`.

Example configuration:

- Target type: Instances
- Protocol: HTTP
- Port: 80
- VPC: `three-tier-vpc`
- Target: The private application EC2 instance

Configure the health check path as `/` if the application returns a successful response there.

**Explanation**

The target group defines the backend resources to which the ALB sends traffic.

The target group uses port 80 because Nginx listens on port 80 on the EC2 host.

The ALB does not need to communicate directly with Flask on port 8000. Nginx forwards requests to Flask inside the Docker network.

The target must pass its health checks to be considered healthy by the ALB.

## 20. Create the Application Load Balancer

Create an Application Load Balancer with these settings:

- Scheme: Internet-facing
- IP address type: IPv4
- VPC: `three-tier-vpc`
- Subnets: `public-a` and `public-b`
- Security group: `alb-sg`
- Listener: HTTP port 80
- Default forwarding action: `three-tier-tg`

**Explanation**

The ALB is the public entry point for the application.

It accepts incoming HTTP requests and forwards them to healthy targets in the target group.

Using public subnets in two Availability Zones allows the load balancer to operate across both zones.

However, if the project has only one EC2 application instance, the application itself does not have full multi-AZ redundancy. For high availability, deploy additional application instances across both Availability Zones.

After the load balancer becomes active, copy its DNS name and open it in a browser to test the application.

## 21. Verify the Application

### Check the containers

```bash
docker-compose ps
```

Confirm that the Nginx and Flask containers are running.

### Check the local HTTP endpoint

```bash
curl -i http://localhost/
```

This tests the Nginx endpoint on the EC2 host.

### Check the target group

Open EC2 → Target Groups → `three-tier-tg`.

Verify that the EC2 target is healthy.

### Check the ALB

Open the ALB DNS name in a browser.

The application should return the four sample blog posts if the database is reachable and the Flask route completes successfully.

### Check database connectivity

From the private application environment, verify that the RDS endpoint is reachable on port 3306 and that the configured database credentials are valid.

Use the RDS endpoint from the console rather than copying a machine-specific endpoint into public documentation.

## 22. Configure CloudWatch Logs

Create CloudWatch log groups for the application components.

Example names:

```text
/three-tier/nginx
/three-tier/flask
```

Set a retention period appropriate for the project. The learning setup uses one week.

**Explanation**

CloudWatch Logs stores log events centrally, allowing administrators to investigate application failures and operational issues.

The Nginx log group can contain access and error logs.

The Flask log group can contain backend application output and errors.

Creating a log group alone does not automatically send container logs to CloudWatch. Configure a supported CloudWatch logging agent or an appropriate Docker logging driver and log delivery mechanism.

### IAM permissions for logging

The EC2 role needs appropriate permissions to create log streams and publish events to the configured log groups.

Example permission actions include:

```text
logs:CreateLogStream
logs:PutLogEvents
logs:DescribeLogStreams
```

Scope the policy to the required log groups in the correct AWS account and region.

Follow least privilege and avoid wildcard permissions when narrower resource permissions are practical.

## 23. Configure CloudWatch Alarms

Create a CloudWatch alarm to monitor the health of the application's load balancer targets.

Example metric:

- Namespace: `AWS/ApplicationELB`
- Metric: `UnHealthyHostCount`
- Dimensions: The correct load balancer and target group
- Threshold: Greater than or equal to 1
- Notification: An SNS topic

**Explanation**

The `UnHealthyHostCount` metric reports the number of unhealthy registered targets.

An alarm can change state when the metric crosses the configured threshold for the specified evaluation periods.

This helps detect application failures, stopped containers, failed health checks, or connectivity problems.

Use the actual load balancer and target group dimensions in the alarm configuration.

For container-level monitoring, an unhealthy target alarm is useful but does not directly prove that a specific Docker container has stopped. Consider additional monitoring if container-level status is required.

## 24. Configure Amazon SNS Email Notifications

Create an SNS topic for application alerts.

Example topic name:

```text
three-tier-alerts
```

Create an email subscription using an address you control and confirm the subscription from the email sent by SNS.

**Explanation**

Amazon SNS distributes messages to subscribed endpoints.

When the CloudWatch alarm transitions to the configured state, SNS can send an email notification to the confirmed subscription.

This makes it easier to detect and respond to operational issues.

Test the integration by deliberately stopping a container in the development environment and checking whether the load balancer target becomes unhealthy and the configured alarm sends a notification.

Allow time for health checks, metric publication, alarm evaluation, and notification delivery. An immediate email is not guaranteed.

## 25. Security Best Practices

The project demonstrates several important cloud security principles.

1. **Private application server:** The EC2 instance has no public IP address.
2. **Private database:** RDS is configured without public access.
3. **Security group restrictions:** Only the ALB security group should reach the Nginx host port, and only the application security group should reach MySQL port 3306.
4. **IAM roles:** AWS permissions are assigned to the EC2 role instead of embedding AWS access keys in the application.
5. **Session Manager:** Administrative access is provided without requiring a publicly exposed SSH port.
6. **Secret management:** Database passwords are excluded from source control.
7. **Centralized logs:** CloudWatch helps with operational troubleshooting.
8. **Monitoring and alerts:** CloudWatch alarms and SNS improve visibility into failures.
9. **HTTPS:** For a production deployment, use TLS certificates and an HTTPS listener on the ALB.
10. **Least privilege:** Limit IAM and database permissions to the actions each component needs.

## 26. Troubleshooting Guide

### Problem: EC2 instance is unavailable in Session Manager

Check the following:

- The SSM agent is installed and running.
- The EC2 instance has the correct IAM role.
- The instance has outbound connectivity to Systems Manager endpoints.
- Required VPC endpoints and endpoint security groups are correctly configured, if used.

### Problem: ALB target is unhealthy

Check:

- The target group port is 80.
- Nginx is running and listening on port 80.
- The application security group allows port 80 from `alb-sg`.
- The health check path returns a successful response.
- The application can reach RDS successfully.

### Problem: Flask cannot connect to RDS

Check:

- The RDS endpoint is correct.
- The database is available.
- The application security group is authorized by `db-sg`.
- The database name, username, and password are correct.
- Port 3306 is reachable from the application environment.
- The required Python MySQL connector is installed.

### Problem: Nginx returns a gateway error

Check:

- The Flask container is running.
- Flask listens on port 8000.
- Nginx uses `proxy_pass http://backend:8000;`.
- Both containers are connected to the same Docker network.
- The Docker Compose service name matches `backend`.

### Problem: CloudWatch logs are empty

Check:

- The application writes logs to a location that the logging mechanism collects.
- The logging agent or Docker logging driver is configured correctly.
- The EC2 role has the necessary CloudWatch permissions.
- The log group name and AWS region are correct.

### Problem: SNS email notification is not received

Check:

- The email subscription is confirmed.
- The alarm has transitioned to the configured state.
- The alarm action references the correct SNS topic.
- The metric dimensions and threshold are correct.

## 27. Architecture Limitations and Future Improvements

This project is a learning implementation and can be improved further.

- Deploy multiple application instances across two Availability Zones.
- Configure an Auto Scaling group for the application tier.
- Use a Multi-AZ RDS deployment for improved database availability.
- Configure HTTPS with an appropriate certificate.
- Use AWS Secrets Manager for database credentials.
- Use a production-grade WSGI server for Flask.
- Add application health-check endpoints.
- Improve CloudWatch metrics, dashboards, and log-based alerts.
- Add automated image builds and deployment through a CI/CD pipeline.
- Use infrastructure as code, such as AWS CloudFormation or Terraform.
- Consider a separate NAT Gateway for each Availability Zone where availability requirements justify the cost.

The logical architecture has three tiers, but the current deployment uses a single application EC2 instance and a Single-AZ RDS database. The design should not be described as fully highly available until those components have appropriate redundancy.

## 28. What I Learned

Through this project, I developed practical understanding of:

- AWS VPC networking and subnet design.
- Route tables, Internet Gateways, and NAT Gateways.
- Application Load Balancer configuration.
- EC2 deployment and private instance administration.
- IAM roles and permission policies.
- Docker and Docker Compose.
- Nginx reverse proxy configuration.
- Flask backend development.
- Amazon RDS MySQL connectivity.
- Security group design and network isolation.
- CloudWatch logging and metric-based alarms.
- SNS email notification integration.
- Troubleshooting a multi-component cloud application.

## 29. Conclusion

The AWS Three-Tier Web Application project demonstrates how networking, compute, application, database, security, and monitoring services can be integrated into a cloud-based architecture.

The project helped me understand how to isolate application components, control network access, manage a containerized application, connect to a private managed database, and configure monitoring and alerts.

It also provides a foundation for further learning in AWS Cloud Engineering, DevOps, infrastructure automation, and production cloud architecture.

## 30. Author

**Project:** AWS Three-Tier Web Application Architecture

**Core services:** Amazon VPC, EC2, Application Load Balancer, RDS for MySQL, IAM, Systems Manager, CloudWatch, SNS, Docker, Nginx, and Flask.

This repository documents a hands-on learning project. Deployment details and architecture should be adapted to the requirements of each AWS environment.

---

## Disclaimer

This project is intended for educational purposes. AWS resources can incur charges, including EC2 instances, load balancers, NAT Gateways, and RDS databases. Review AWS pricing and remove unused resources after testing.
