---

# Dynamic Web Application with Nginx, Node.js, and SSL using Let's Encrypt

This document outlines the process of building a dynamic web application using GCP, Node.js, and Nginx, highlighting the challenges encountered and solutions implemented throughout the project.

Table of Contents

1. [Project Overview](#project-overview)
2. [Provisioning the Virtual Machine on GCP](#provisioning-the-virtual-machine-on-gcp)
3. [Installing Node.js and Nginx](#installing-nodejs-and-nginx)
4. [Setting Up the Web Application](#setting-up-the-web-application)
5. [Configuring Reverse Proxy with Nginx](#configuring-reverse-proxy-with-nginx)
6. [Using No-IP for Dynamic DNS](#using-no-ip-for-dynamic-dns)
7. [Installing and Configuring Let's Encrypt SSL](#installing-and-configuring-lets-encrypt-ssl)
8. [Final Testing and Verification](#final-testing-and-verification)
9. [Troubleshooting](#troubleshooting)
10. [Conclusion](#conclusion)

---

## Project Overview

This project demonstrates how to create a **dynamic landing page** hosted on a **Google Cloud Platform (GCP)** Virtual Machine (VM), powered by **Node.js** for the backend, with **Nginx** as a reverse proxy. The project is secured using **SSL certificates** from **Let’s Encrypt**. Access to the web application is facilitated via a **No-IP subdomain** (`luu.ddns.net`), pointing to the public IP of the VM. The task also includes configuring security with **HTTPS** and deploying the project using professional standards.

---

## Provisioning the Virtual Machine on GCP

### Steps Taken:

1. **Creating the GCP VM Instance**:

   * I created a **VM instance** in the **GCP Console** with **Ubuntu 22.04 LTS**.
   * The instance uses the **e2-micro** machine type (eligible for the free-tier).
   * Allocated **10 GB** for the boot disk and enabled **HTTP** and **HTTPS** traffic in the firewall rules.
   * The **public IP address** (`34.27.120.7`) was assigned to the VM upon creation.

2. **Error Encountered**:

   * During VM creation, the VM failed to start due to missing **permissions** for the **Compute Engine API**.
   * **Resolution**: I enabled the **Compute Engine API** in **IAM & Admin** and retried the process.

3. **Connecting to the VM**:

   * Once the VM was provisioned, I accessed it using **SSH** from the GCP console:

     ```bash
     ssh [luu]@34.27.120.7
     ```

---

## Installing Node.js and Nginx

### Steps Taken:

1. **Installing Node.js**:

   * After connecting to the VM, I installed **Node.js** (version 18) and **npm** to run the backend application:

     ```bash
     sudo apt update
     sudo apt install nodejs npm -y
     ```

2. **Installing Nginx**:

   * I installed **Nginx** to handle incoming web requests and proxy them to the Node.js application:

     ```bash
     sudo apt install nginx -y
     ```

3. **Error Encountered**:

   * **Issue**: Nginx wasn’t starting due to blocked ports. I could not access the server on HTTP (port 80) and HTTPS (port 443).
   * **Resolution**: I resolved this by configuring **UFW** (Uncomplicated Firewall) to allow traffic on **port 80** and **443**:

     ```bash
     sudo ufw allow 'Nginx Full'
     sudo systemctl restart nginx
     ```

4. **Verification**:

   * I confirmed that **Nginx** was running with:

     ```bash
     sudo systemctl status nginx
     ```

---

## Setting Up the Web Application

### Steps Taken:

1. **Create the Application Directory**:

   * I created the project directory for **Node.js**:

     ```bash
     mkdir ~/app
     cd ~/app
     ```

2. **Initialize Node.js Project and Install Express**:

   * I initialized the project using **npm** and installed **Express**:

     ```bash
     npm init -y
     npm install express
     ```

3. **Creating `server.js` for Node.js**:

   * I created a basic **Node.js server** that listens on port **3000** and serves the static landing page:

     ```javascript
     const express = require('express');
     const app = express();

     // Serve static files
     app.use(express.static('public'));

     // Main route
     app.get('/', (req, res) => {
         res.sendFile(__dirname + '/public/index.html');
     });

     // Start server
     app.listen(3000, () => {
         console.log('Server running at http://localhost:3000');
     });
     ```

4. **Creating `index.html`**:

   * I created the **landing page** using **Tailwind CSS**:

    ## see index.html file

---

## Configuring Reverse Proxy with Nginx

### **Key Issue Encountered: Using IP Address in `server_name` Directive**

1. **Initial Configuration with IP Address**:

   * Initially, I configured Nginx with my **public IP** (`34.27.120.7`) in the **`server_name`** directive:

     ```nginx
     server {
         listen 80;
         server_name 34.27.120.7;  # Using the public IP address here

         location / {
             proxy_pass http://127.0.0.1:3000;
             proxy_http_version 1.1;
             proxy_set_header Upgrade $http_upgrade;
             proxy_set_header Connection 'upgrade';
             proxy_set_header Host $host;
             proxy_cache_bypass $http_upgrade;
         }
     }
     ```

2. **Error Encountered**:

   * **Issue**: Nginx failed to start, and I received an error indicating that **server\_name** was set to an **IP address**, which **Let’s Encrypt** does not support for SSL certificates. Using an IP address for SSL and virtual hosts can cause issues.

     * **Error Message**: `nginx: [emerg] "server_name" directive is not allowed here in /etc/nginx/sites-enabled/default:80`
     * **Resolution**: I replaced the **IP address** with a **valid domain name** (`luu.ddns.net`) in the **`server_name`** directive:

       ```nginx
       server {
           listen 80;
           server_name luu.ddns.net;

           location / {
               proxy_pass http://127.0.0.1:3000;
               proxy_http_version 1.1;
               proxy_set_header Upgrade $http_upgrade;
               proxy_set_header Connection 'upgrade';
               proxy_set_header Host $host;
               proxy_cache_bypass $http_upgrade;
           }
       }
       ```

3. **Test and Reload Nginx**:

   * After making the change, I tested the configuration and reloaded Nginx:

     ```bash
     sudo nginx -t
     sudo systemctl reload nginx
     ```

---

## Using No-IP for Dynamic DNS

### Steps Taken:

1. **Create a Free Subdomain on No-IP**:

   * I registered for a **No-IP** account and created the free subdomain (`luu.ddns.net`).

2. **Update DNS to Point to Public IP**:

   * I configured the **DNS settings** in **No-IP** to point to my **public IP** (`34.27.120.7`).

3. **Error Encountered**:

   * **Issue**: DNS propagation was delayed.
   * **Resolution**: I used **DNSChecker** to verify DNS resolution and waited for propagation.

---

## Installing and Configuring Let's Encrypt SSL

### Steps Taken:

1. **Install Certbot**:

   * I installed **Certbot** and its **Nginx plugin**:

     ```bash
     sudo apt install certbot python3-certbot-nginx -y
     ```

2. **Obtain and Install SSL Certificate**:

   * I ran **Certbot** to obtain and install the **SSL certificate** for `luu.ddns.net`:

     ```bash
     sudo certbot --nginx -d luu.ddns.net
     ```

3. **Verification**:

   * After successful installation, I confirmed that the website was **secure** by visiting **`https://luu.ddns.net`** and seeing the **green padlock**.

---

## Final Testing and Verification

1. **Verify SSL**:

   * I visited **`https://luu.ddns.net`** to ensure the page was served over **HTTPS** and checked the SSL certificate.

2. **Verify Node.js App**:

   * The **Node.js** app was successfully served over **HTTPS** through **Nginx**.

3. **SSL Labs SSL Test**:

   * I ran the **SSL Labs’ SSL Test** to verify the SSL configuration and certificate.

---

## Troubleshooting

1. **Certbot Issues**:

   * **Issue**: Certbot could not find the matching server block because the **`server_name`** directive was initially set to the **IP address**.
   * **Resolution**: I fixed the issue by changing **`server_name`** to `luu.ddns.net`.

2. **Nginx Errors**:

   * **Issue**: Nginx couldn’t start with the **IP address** in the **`server_name`** directive.
   * **Resolution**: I replaced the **IP address** with the **domain name** (`luu.ddns.net`) and reloaded Nginx.

3. **Nginx Failing to Start**:

   * I ran **`sudo nginx -t`** to check for configuration syntax errors and reloaded Nginx once fixed.

---

## Conclusion

This project demonstrates how to set up a **dynamic landing page** hosted on **GCP**, with **Node.js** as the backend, **Nginx** as the reverse proxy, and **SSL** from **Let’s Encrypt**. **No-IP** was used to point a free subdomain (`luu.ddns.net`) to my public IP, and SSL certificates were correctly installed to secure the application.

---

[Check out the live site](https://luu.ddns.net)


## Prerequisites

- GCP Account (free tier eligible)
- Domain access (No-IP account)
- Basic knowledge of:
  - Linux commands
  - Node.js
  - Nginx

---

### Screenshot of the Rendered Page:

![Rendered Page Screenshot](screenshot.png)

---
