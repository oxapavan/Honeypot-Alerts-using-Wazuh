
# Building a Robust SIEM on AWS with Wazuh: A Detailed Guide

Building a SIEM solution is crucial for any organization looking to enhance its security posture. Wazuh, an open-source security platform, offers a comprehensive suite of features for threat detection, monitoring, incident response, and compliance. Deploying Wazuh on Amazon Web Services (AWS) leverages the scalability, flexibility, and reliability of cloud infrastructure to create a powerful and efficient SIEM.

---

## Why Wazuh on AWS?

Deploying Wazuh on AWS combines the strengths of both platforms:

* **Scalability:** Easily scale your Wazuh environment up or down based on your needs, without significant upfront hardware investments.
* **Flexibility:** Leverage various AWS services like EC2, S3, and CloudTrail for seamless integration and enhanced security monitoring.
* **Reliability:** Benefit from AWS's robust infrastructure, ensuring high availability and durability for your SIEM solution.
* **Cost-Effectiveness:** Pay only for the resources you consume, optimizing your operational costs.
* **Centralized Monitoring:** Collect and analyze logs from your AWS infrastructure (EC2 instances, S3 buckets, CloudTrail logs, etc.) in a centralized location, providing a holistic view of your security events.

---

## Prerequisites

Before you begin, ensure you have the following:

* **An AWS Account:** With sufficient permissions to launch EC2 instances, create S3 buckets, and configure IAM roles/users.
* **Basic AWS Knowledge:** Familiarity with EC2, VPC, Security Groups, and IAM concepts.
* **SSH Client:** To connect to your Wazuh instance.
---

## Step-by-Step Deployment using AWS Marketplace AMI

The easiest way to deploy Wazuh on AWS is by using the **Wazuh All-In-One Deployment AMI** available in the AWS Marketplace. This AMI comes pre-configured with the Wazuh Manager, Wazuh Indexer (OpenSearch), and Wazuh Dashboard, simplifying the setup process significantly.

### Step 1: Launch the Wazuh All-In-One AMI

1.  **Navigate to AWS Marketplace:** Log in to your AWS Management Console and search for "Wazuh All-In-One Deployment" in the AWS Marketplace.
    
2.  **Subscribe and Continue:** Click on the "Wazuh All-In-One Deployment" listing, review the details, and click **Continue to Subscribe**. Accept the terms and then click **Continue to Configuration**.
3.  **Choose Software Version and Region:** Select your desired **Software Version** and the **AWS Region** where you want to deploy Wazuh. Click **Continue to Launch**.
4.  **Launch the Instance:**
    * **EC2 Instance Type:** Wazuh recommends using an instance type like `c5a.xlarge` or larger for production environments. For testing or small environments, `t2.medium` or `t2.large` might suffice (minimum 2 CPU, 4 GiB RAM recommended).
    * **VPC and Subnet:** Choose the VPC and subnet where you want to deploy your instance.
    * **Security Group:** Select an existing security group or **Create new based on seller settings**. The latter will create a new security group with the necessary ports (e.g., 22 for SSH, 443 for HTTPS, 1514 for agent communication) pre-configured. If creating manually, ensure you open ports crucial for Wazuh operation.
    * **Key Pair:** Choose an existing key pair or create a new one. This key pair is essential for SSH access to your instance.
    * **Storage:** The AMI typically defaults to 100 GiB GP3 or more. You can adjust this based on your expected log volume.
    * **Review and Launch:** Review all your configurations and then click **Launch**.

---

### Step 2: Initial Access and Dashboard Login

Once your EC2 instance is in the "Running" state (this might take a few minutes):

1.  **Get Instance ID:** Go to the EC2 console, select your newly launched Wazuh instance, and copy its **Instance ID** (e.g., `i-07f25f6afe4789342`).
    **Important:** The default passwords for Wazuh Indexer users (including `admin`) are automatically set to the capitalized instance ID.
2.  **Access the Wazuh Dashboard:** Open your web browser and navigate to `https://<Your_Instance_Public_IPv4_Address>`.
    * You might encounter a certificate warning; you can proceed to the site.
    * **Login Credentials:**
        * **Username:** `admin`
        * **Password:** Your **Instance ID** (e.g., `I-07f25f6afe4789342` - note the capitalization of the first letter).
    

---

### Step 3: Integrating with AWS CloudTrail (Basic Setup)

To monitor activities within your AWS infrastructure, you can integrate Wazuh with AWS CloudTrail.

1.  **Enable AWS CloudTrail:**
    * In the AWS Console, navigate to **CloudTrail**.
    * Click **Create trail**.
    * Configure the trail:
        * **Trail name:** e.g., `wazuh-cloudtrail-logs`
        * **Storage location:** Choose to create a new S3 bucket or select an existing one where CloudTrail logs will be stored (e.g., `my-wazuh-cloudtrail-bucket`).
        * **Log file validation:** Enable this for integrity checks.
        * **Event types:** Select **Management events** and optionally **Data events** for deeper visibility.
        * Follow the prompts to create the trail. Consider enabling **Multi-Region CloudTrail** for comprehensive visibility across all AWS regions.
2.  **Create an IAM User for Wazuh (or Role):**
    * Create an IAM user with programmatic access.
    * Attach a policy that grants read-only access to the S3 bucket where your CloudTrail logs are stored (e.g., `s3:GetObject`, `s3:ListBucket` on your specific bucket).
    * Generate the Access Key ID and Secret Access Key. **Save these securely**, as you'll need them for Wazuh configuration.
3.  **Configure Wazuh to Collect CloudTrail Logs:**
    * **SSH into your Wazuh instance:** Use the key pair you selected during launch and the `wazuh-user` (e.g., `ssh -i "your-key.pem" wazuh-user@<Your_Instance_Public_IPv4_Address>`).
    * **Install AWS Module (if not already present):**
        ```bash
        sudo apt update
        sudo apt install wazuh-integrations # For Debian/Ubuntu
        # Or for RHEL/CentOS: sudo yum install wazuh-integrations
        ```
    * **Configure AWS Credentials:**
        Create the `~/.aws` directory and `credentials` file if they don't exist:
        ```bash
        mkdir -p /root/.aws
        nano /root/.aws/credentials
        ```
        Add your AWS credentials (using a profile, e.g., `default`):
        ```ini
        [default]
        aws_access_key_id = YOUR_ACCESS_KEY_ID
        aws_secret_access_key = YOUR_SECRET_ACCESS_KEY
        region = YOUR_AWS_REGION
        ```
        Save and exit.
    * **Edit Wazuh Manager Configuration:**
        Open the Wazuh manager configuration file:
        ```bash
        sudo nano /var/ossec/etc/ossec.conf
        ```
        Add the following `wodle` block within the `<ossec_config>` section, replacing `my-wazuh-cloudtrail-bucket` with your S3 bucket name:
        ```xml
        <wodle name="aws-s3">
          <disabled>no</disabled>
          <interval>30m</interval>
          <run_on_start>yes</run_on_start>
          <skip_on_error>no</skip_on_error>
          <bucket type="cloudtrail">
            <name>my-wazuh-cloudtrail-bucket</name>
            <aws_profile>default</aws_profile>
          </bucket>
        </wodle>
        ```
        You can add multiple `<bucket>` entries for different log types or buckets.
        Save the file and exit.
    * **Restart Wazuh Manager:**
        ```bash
        sudo systemctl restart wazuh-manager
        ```
4.  **Verify Log Collection:**
    * Wait a few minutes for logs to start flowing.
    * In the Wazuh Dashboard, navigate to **Security Events** and search for `aws.cloudtrail`. You should start seeing events from your AWS CloudTrail.
    * You can also check the Wazuh manager logs for the AWS module:
        ```bash
        tail -f /var/ossec/logs/ossec.log | grep aws
        ```
    

---

## Conclusion

By following these steps, you can successfully deploy a Wazuh SIEM solution on AWS using the All-In-One AMI and integrate it with AWS CloudTrail to start monitoring your cloud environment. This setup provides a robust foundation for proactive threat detection, security analysis, and compliance in your AWS infrastructure. Remember to explore Wazuh's extensive capabilities for agent deployment on EC2 instances, file integrity monitoring, vulnerability detection, and more to fully leverage its potential.
````
