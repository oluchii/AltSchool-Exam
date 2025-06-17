---

# Dynamic Web Application with Nginx, Node.js, and SSL using Let's Encrypt

This document outlines the process of building a dynamic web application using **Google Cloud Platform (GCP)**, **Node.js**, and **Nginx**, highlighting the challenges encountered and solutions implemented throughout the project.

---

## Table of Contents

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

This project demonstrates the process of creating a **dynamic landing page** hosted on a **Google Cloud Platform (GCP)** Virtual Machine (VM), with **Node.js** powering the backend and **Nginx** used as the reverse proxy. The project is secured with **SSL certificates** from **Let’s Encrypt**, and access is facilitated through a **No-IP subdomain** (`luu.ddns.net`), which points to the public IP of the VM. The task also includes configuring **HTTPS** and deploying the project in a **production-ready** manner.

---

## Provisioning the Virtual Machine on GCP

### Steps Taken:

1. **Creating the GCP VM Instance**:

   * A **VM instance** was created in the **GCP Console** using **Ubuntu 22.04 LTS**.
   * The **e2-micro** machine type was selected (eligible for the free-tier).
   * **10 GB** was allocated for the boot disk, and both **HTTP** and **HTTPS** traffic were allowed through the firewall rules.
   * A **public IP address** (`34.27.120.7`) was assigned to the VM upon creation.

2. **Error Encountered**:

   * The **VM failed to start** due to missing **permissions** for the **Compute Engine API**.
   * **Resolution**: The **Compute Engine API** was enabled in the **IAM & Admin** section of GCP, and the VM creation process was retried.

3. **Connecting to the VM**:

   * SSH access was established through the GCP console with the following command:

     ```bash
     ssh [luu]@34.27.120.7
     ```

---

## Installing Node.js and Nginx

### Steps Taken:

1. **Installing Node.js**:

   * **Node.js** (version 18) and **npm** were installed to run the backend application:

     ```bash
     sudo apt update
     sudo apt install nodejs npm -y
     ```

2. **Installing Nginx**:

   * **Nginx** was installed to handle incoming web requests and proxy them to the Node.js application:

     ```bash
     sudo apt install nginx -y
     ```

3. **Error Encountered**:

   * **Issue**: Nginx failed to start due to blocked ports. The server was inaccessible on HTTP (port 80) and HTTPS (port 443).
   * **Resolution**: The firewall (UFW) was configured to allow traffic on **port 80** and **443**:

     ```bash
     sudo ufw allow 'Nginx Full'
     sudo systemctl restart nginx
     ```

4. **Verification**:

   * The status of Nginx was checked to ensure it was running:

     ```bash
     sudo systemctl status nginx
     ```

---

## Setting Up the Web Application

### Steps Taken:

1. **Create the Application Directory**:

   * A directory was created for the **Node.js** project:

     ```bash
     mkdir ~/app
     cd ~/app
     ```

2. **Initialize Node.js Project and Install Express**:

   * A new **Node.js** project was initialized, and **Express** was installed:

     ```bash
     npm init -y
     npm install express
     ```

3. **Creating `server.js` for Node.js**:

   * A basic **Node.js server** was created, which listens on port **3000** and serves the static landing page:

     ```javascript
     const express = require('express');
     const app = express();

     app.use(express.static('public'));

     app.get('/', (req, res) => {
         res.sendFile(__dirname + '/public/index.html');
     });

     app.listen(3000, () => {
         console.log('Server running at http://localhost:3000');
     });
     ```

4. **Creating `index.html`**:

   * The **landing page** was designed using **Tailwind CSS**. The page includes information such as the project title, a short pitch, and a professional bio.

   ### See `index.html` file for full content

---

## Configuring Reverse Proxy with Nginx

### **Key Issue Encountered: Using IP Address in `server_name` Directive**

1. **Initial Configuration with IP Address**:

   * Initially, the **public IP** (`34.27.120.7`) was used in the **`server_name`** directive in the Nginx configuration:

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

   * **Issue**: Nginx failed to start due to the use of the **IP address** in the **`server_name`** directive. **Let’s Encrypt** does not allow SSL certificates to be issued for **IP addresses**.

     * **Error Message**: `nginx: [emerg] "server_name" directive is not allowed here in /etc/nginx/sites-enabled/default:80`
   * **Resolution**: The **`server_name`** was updated to use a valid **domain name** (`luu.ddns.net`) instead of the IP address. The updated configuration was as follows:

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

   * After making the necessary change, the configuration was tested for errors:

     ```bash
     sudo nginx -t
     sudo systemctl reload nginx
     ```

---

## Using No-IP for Dynamic DNS

### Steps Taken:

1. **Create a Free Subdomain on No-IP**:

   * A **No-IP** account was created, and the subdomain **`luu.ddns.net`** was registered.

2. **Update DNS to Point to Public IP**:

   * The **DNS settings** in **No-IP** were configured to point to the **public IP** (`34.27.120.7`).

3. **Error Encountered**:

   * **Issue**: DNS propagation took some time.
   * **Resolution**: The **DNSChecker** tool was used to verify that DNS resolution was correct and the changes were allowed to propagate.

---

## Installing and Configuring Let's Encrypt SSL

### Steps Taken:

1. **Install Certbot**:

   * **Certbot** was installed along with the **Nginx plugin**:

     ```bash
     sudo apt install certbot python3-certbot-nginx -y
     ```

2. **Obtain and Install SSL Certificate**:

   * The **SSL certificate** was obtained and installed for the **`luu.ddns.net`** domain using **Certbot**:

     ```bash
     sudo certbot --nginx -d luu.ddns.net
     ```

3. **Verification**:

   * After the SSL certificate was successfully installed, the website was tested for **HTTPS** access by visiting **`https://luu.ddns.net`** and confirming the presence of the **green padlock** indicating that the connection was secure.

---

## Final Testing and Verification

### Steps Taken:

1. **Verify SSL**:

   * The **SSL certificate** was verified by visiting **`https://luu.ddns.net`**.

2. **Verify Node.js App**:

   * The **Node.js** application was successfully served over **HTTPS** via **Nginx**.

3. **SSL Labs SSL Test**:

   * The **SSL Labs SSL Test** was used to verify the configuration of the SSL certificate, ensuring proper installation.

---

## Troubleshooting

1. **Certbot Issues**:

   * **Issue**: Certbot couldn’t find the matching server block because the **`server_name`** directive was initially set to the **IP address**.
   * **Resolution**: The **server\_name** directive was updated to **`luu.ddns.net`** in the Nginx configuration, which resolved the issue.

2. **Nginx Errors**:

   * **Issue**: Nginx failed to start with the **IP address** in the **`server_name`** directive.
   * **Resolution**: Replaced the **IP address** with the **domain name** (`luu.ddns.net`) in the **server\_name** directive, followed by reloading Nginx.

3. **Nginx Failing to Start**:

   * **Solution**: Ran `sudo nginx -t` to check for configuration issues and reloaded Nginx after fixing the errors.

---

## Conclusion

This project demonstrates how to set up a **dynamic landing page** hosted on **GCP**, with **Node.js** as the backend, **Nginx** as the reverse proxy, and **SSL** from **Let’s Encrypt**. **No-IP** was used to point the **subdomain** (`luu.ddns.net`) to the **public IP**, ensuring secure access to the application.

---

### Screenshot of the Rendered Page:

![Rendered Page Screenshot](screenshot.png)

---