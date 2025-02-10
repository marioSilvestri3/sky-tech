# **Project: Deploying a .NET App on AWS**
## **Objective:**  
Deploy a .NET app on AWS, including setting up Windows Server, IIS, and SQL Server; and implementing best practices for security and performance.

---

## **Phase 0: Project Research and Planning**
- Consulted with ChatGPT (4o and o3-mini-high) on project design and implementation.
- Created Github repository
- Light research on Microsoft ASP.NET Architecture
    - https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures
- Research on .NET on AWS at Tyler
    - https://aws.amazon.com/blogs/modernizing-with-aws/tyler-tech-improved-access-to-justice-during-pandemic/
    - https://samnewman.io/patterns/architectural/bff/
- Open Source at Tyler
    -https://github.com/tyler-technologies-oss
    
---

## **Phase 1: AWS Account & Infrastructure Setup**
### **1.1 AWS Account Setup**
1. **Create an AWS Account**
2. **Enable MFA**
3. **Set up Billing Alerts**
4. **IAM User** with **AdministratorAccess** for security instead of using the root account.
   - Setup AWS Organization and enabled IAM Identity Center
   - Created admin user

---

### **1.2 Create AWS Infrastructure**
1. **Launch an EC2 Windows Server Instance**
   - Researching Windows Server versions
   - Researching EC2 size and best practices for IIS + SQL
      - https://docs.aws.amazon.com/prescriptive-guidance/latest/optimize-costs-microsoft-workloads/sql-server.html
      - https://docs.aws.amazon.com/codedeploy/latest/userguide/tutorials-windows.html
      - https://docs.aws.amazon.com/launchwizard/latest/userguide/launch-wizard-iis.html
      - https://docs.aws.amazon.com/launchwizard/latest/userguide/launch-wizard-iis-getting-started.html
   - Researching deployment strategies
      - Manual, OpsWorks (deprecated), Launch Manager, Systems Manager, CodeDeploy
   - Researching VPC
      - https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html
      - http://www.faqs.org/rfcs/rfc1918.html
      - https://en.wikipedia.org/wiki/Private_network
      - https://medium.com/aws-activate-startup-blog/practical-vpc-design-8412e1a18dcc
      - https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/welcome.html
   - Created VPC
      - 10.0.0.0/16
      - 2 AZ
      - 4 /20 Subnets
      - 1 Private & 1 Public per AZ
      - 1 Public NAT per AZ
      - 1 IGW
      - 1 Gateway endpoint to S3 (Might use this for SQL Server backups)

   - **AMI:** Windows Server 2022 Base
   - **Instance Type:** t3.medium (for reasonable performance)
   - **Key Pair:** Create and download one
   - **Security Group:** Allow RDP (port 3389) from your IP, and HTTP/HTTPS (80, 443)
   - **Elastic IP:** Assign a static IP for convenience

🔹 **Optional Step:**  
Instead of using RDP, enable **AWS Session Manager** (IAM Role + SSM Agent) for more secure access.

2. **Create an RDS SQL Server Instance**
   - AWS Console → RDS → Create Database → SQL Server
   - **DB Engine:** SQL Server Express (for cost efficiency)
   - **Instance Class:** db.t3.medium
   - **Storage:** 20GB SSD
   - **Security Group:** Allow inbound SQL connections (port 1433) from the EC2 instance

🔹 **Optional Step:**  
Set up a **Windows EC2 with SQL Server manually** instead of using RDS for full control.

---

## **Phase 2: Windows Server Setup**
### **2.1 Connect to EC2 Instance**
- RDP into your instance using the public IP (from Elastic IP)
- Run `sconfig` to:
  - **Set a static private IP**
  - **Enable Windows Updates**
  - **Change computer name** (e.g., `WEB-SERVER-01`)
  - **Add an administrator password**

---

### **2.2 Install IIS (Internet Information Services)**
#### **Manual Steps**
1. **Open Server Manager** → Add Roles and Features
2. Select **Web Server (IIS)** and install with default options.

#### **PowerShell Alternative**
```powershell
Install-WindowsFeature -name Web-Server -IncludeManagementTools
```

3. **Verify IIS is Running**  
   - Open `http://localhost` in a browser inside the instance.
   - If successful, you’ll see the default IIS welcome page.

🔹 **Optional Step:**  
Configure **custom error pages, logs, and performance optimizations** in IIS Manager.

---

## **Phase 3: SQL Server Setup & Configuration**
### **3.1 Connect to SQL Server**
1. Open **SQL Server Management Studio (SSMS)**  
2. **Connect to RDS Endpoint** using SQL authentication  
   - **Server Name:** `<your-RDS-endpoint>.rds.amazonaws.com`
   - **Authentication:** SQL Server Authentication
   - **Username/Password:** As set during RDS creation  

🔹 **Optional Step:**  
Enable **Windows Authentication** for AD integration.

---

### **3.2 Create a Database**
```sql
CREATE DATABASE MyWebApp;
USE MyWebApp;
CREATE TABLE Users (
    ID INT PRIMARY KEY IDENTITY(1,1),
    Name NVARCHAR(100),
    Email NVARCHAR(100)
);
INSERT INTO Users (Name, Email) VALUES ('John Doe', 'john@example.com');
```

---

### **3.3 Configure SQL Server for Remote Access**
1. **Allow SQL Server Authentication Mode**
   ```sql
   EXEC sp_configure 'remote access', 1;
   RECONFIGURE;
   ```
2. **Modify AWS Security Group** to allow **port 1433** from your EC2.

🔹 **Optional Step:**  
Implement **SQL Always On Availability Groups** for high availability.

---

## **Phase 4: Deploy an ASP.NET Application**
### **4.1 Install .NET Hosting Bundle**
1. Download and install `.NET Hosting Bundle` ([Microsoft Download](https://dotnet.microsoft.com/en-us/download))
2. Restart IIS after installation:
   ```powershell
   Restart-Service W3SVC
   ```

---

### **4.2 Deploy the ASP.NET Web App**
1. Copy your ASP.NET app files to `C:\inetpub\wwwroot\MyWebApp`
2. Create an IIS **Application Pool**
   - Name: `MyWebApp`
   - Set to **Integrated Mode**
   - Identity: **ApplicationPoolIdentity**
3. Create a **New IIS Website**
   - Physical Path: `C:\inetpub\wwwroot\MyWebApp`
   - Bindings: `http://*:80`

4. **Restart IIS**
   ```powershell
   iisreset
   ```

🔹 **Optional Step:**  
Deploy an **ASP.NET Core Web App** instead of a standard ASP.NET app.

---

## **Phase 5: Secure & Optimize**
### **5.1 Set Up SSL/TLS**
1. **Create a Self-Signed Certificate**
   ```powershell
   New-SelfSignedCertificate -DnsName "yourdomain.com" -CertStoreLocation Cert:\LocalMachine\My
   ```
2. Bind it in **IIS → Site Bindings → HTTPS**  
3. Restart IIS:
   ```powershell
   iisreset
   ```

🔹 **Optional Step:**  
Use **Let’s Encrypt** for a free CA-issued certificate.

---

### **5.2 Set Up Auto Backups for SQL Server**
1. Create a full database backup:
   ```sql
   BACKUP DATABASE MyWebApp TO DISK = 'C:\backups\MyWebApp.bak' WITH FORMAT;
   ```
2. Schedule a **Windows Task** to run this backup daily.

🔹 **Optional Step:**  
Use **AWS Backup** to automate this process.

---

### **5.3 Optimize Performance**
1. **IIS Tweaks**
   - Enable **Output Caching** for static files
   - Set **Compression** for faster content delivery
   - Tune **App Pool Recycling** to avoid memory leaks

2. **SQL Performance**
   - Add indexing:
     ```sql
     CREATE INDEX idx_users_email ON Users(Email);
     ```
   - Enable **Query Store** for monitoring slow queries.

🔹 **Optional Step:**  
Use **AWS CloudWatch** to monitor EC2 and RDS performance.

---

## **Phase 6: Documentation & Cleanup**
### **6.1 Document the Deployment**
- Take screenshots of **EC2, IIS, SQL Server, and AWS settings**
- Write a **step-by-step installation guide**
- Document **troubleshooting steps** for common errors

### **6.2 Clean Up AWS Resources**
- **Terminate EC2 and RDS** instances if you don’t want ongoing charges.

---

## **Final Notes**
🎯 **Key Learning Takeaways:**
- Hands-on experience with **Windows Server, IIS, SQL Server, AWS networking**
- Basic automation via **PowerShell**
- Security practices **(IAM, SSL, backups)**
- Troubleshooting skills **(SQL connectivity, IIS errors, AWS security groups)**