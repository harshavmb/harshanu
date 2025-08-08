---
author: "Harshanu"
title: "Photoprism on Hetzner Cloud"
date: 2023-08-21
description: "A Guide to Building Your Own PhotoPrism Setup on Hetzner Cloud"
tags: ["photoprism", "hetzner", "cloudflare", "cloud", "nginx", "bigtech", "storagebox", "isp"]
thumbnail: https://photos.harshanu.space/api/v1/t/6d028bdd53ee6208c10751515057942eb8f2e815/2zwabhu7/fit_2048
---

## Introduction

In a world where memories are increasingly captured in pixels, managing and sharing our ever-growing photo collections has become both a challenge and a delight. Whether you're a photography enthusiast, a family historian, or simply someone who loves capturing moments, organizing and accessing your photos should be an effortless experience.

Welcome to our guide, where we unveil the secrets of building a robust and secure PhotoPrism setup that puts you in control of your visual memories. This comprehensive setup combines the power of cloud hosting, containerization, reverse proxy, database management, encryption, and more, all working seamlessly to create an efficient and user-friendly platform for your cherished photographs.

Whether you're a seasoned tech enthusiast looking for an exciting project or someone seeking an elegant solution to their photo management needs, our guide has something for you. Join us on this journey as we explore each component of this setup, from the server to the database, and from SSL encryption to reverse proxy, all culminating in a system that will revolutionize the way you interact with your photos.

### Photoprism
PhotoPrism® is an AI-Powered Photos App for the Decentralized Web. It makes use of the latest technologies to tag and find pictures automatically without getting in your way. You can run it at home, on a private server, or in the cloud. 

### Hetzner
Hetzner is a German hosting company that offers a wide range of hosting services, including dedicated servers, virtual private servers, and cloud computing. Hetzner is a popular choice for hosting websites and applications due to its reliable performance and affordable prices. Hetzner also offers a wide range of features, such as IPv6 support, RAID storage, and DDoS protection.

### Cloudflare
Cloudflare is a content delivery network (CDN) and web application firewall (WAF) that helps to protect websites from attacks and improve their performance. Cloudflare also offers a number of other features, such as SSL certificates, image optimization, and bot mitigation.

## Architecture Overview:

```shell
[Internet]
    |
    |
   Cloudflare
    |
    |
   Hetzner Server
    |
    |
    |-- nginx (Reverse Proxy)
    |
    |-- PhotoPrism (Docker container)
    |
    |-- Mariadb Database
    |
    |-- Storage Box
    |
    |-- Let's Encrypt (for SSL certificates)
```

**Internet**: Represents the end user or client accessing your PhotoPrism setup.

**Cloudflare**: This is your DNS provider and reverse proxy. It protects your Hetzner instance by acting as a security barrier and accelerates content delivery through its global network.

**Hetzner (Server)**: Your server is hosted on Hetzner. It's the core of your setup, where PhotoPrism, Nginx, MySQL, and other components reside.

**Nginx (Reverse Proxy)**: Nginx acts as a reverse proxy server, routing incoming requests to the appropriate services within your server.

**PhotoPrism (Docker Container)**: PhotoPrism runs inside a Docker container on your server. It manages your photo collection and provides a web interface for users.

**Mariadb Database**: Mariadb is used for data persistence. PhotoPrism stores metadata and other information in this database.

**Hetzner (Storage Box)**: Your storage box on Hetzner is where your actual photo media files are stored.

**Let's Encrypt**: This represents the SSL certificate provided by Let's Encrypt. It secures the communication between Cloudflare and your Hetzner instance.

## Flow explanation:
* The user initiates a request, which first goes through Cloudflare.
* Cloudflare serves as a DNS provider, directing the request to the appropriate server.
* It also acts as a reverse proxy, protecting your Hetzner instance from direct exposure to the internet and providing security features.
* The request then reaches your Hetzner server.
* Nginx, configured as a reverse proxy, forwards the request to the appropriate service inside your server.
* PhotoPrism, running in a Docker container, handles photo management and serves the web interface.
* MySQL is used to store and retrieve metadata and other data related to your photo collection.
* Your actual photo files are stored on the Hetzner storage box.
* Let's Encrypt ensures that the communication between Cloudflare and your Hetzner instance is encrypted and secure.

**Tip**: For faster performance you can use `Hetzner Volumes` for thumbnail caching, sidecars etc., But you need to pay more money for Volumes. 
Having deployed Photoprism with over 26,000 photos, I've not seen any problem using `storage box`. The latency between the server & storage box is under `1ms`. This tells the requests from the server to storage box are routed via the Hetzner backbone network. 

## Security Measures:
When it comes to your cherished photo collection, security is paramount. Ensuring the safety and confidentiality of your images, metadata, and personal data should be a top priority. In our PhotoPrism setup, we've implemented a series of robust security measures to protect your digital treasure trove:

1. **Cloudflare Shield**: As your DNS provider and reverse proxy, Cloudflare acts as a powerful shield against various online threats, including DDoS attacks, SQL injections, and more. It ensures that malicious traffic never reaches your Hetzner server.

2. **SSL Encryption with Let's Encrypt**: Communication between Cloudflare and your Hetzner instance is secured using SSL encryption, courtesy of Let's Encrypt. This ensures that data transferred between your users and your server remains private and tamper-proof.

3. **Hetzner Firewall**: Utilize Hetzner's built-in firewall to further enhance security. Configure it to allow only essential traffic, adding an extra layer of defense against unauthorized access.

4. **Regular Software Updates**: Keep all software components, including the server operating system, Nginx, PhotoPrism, and MySQL, up to date with the latest security patches. Vulnerabilities in outdated software can be exploited by malicious actors.

5. **Data Backup and Redundancy**: Regularly backup your photo collection and database. Store backups in secure, offsite locations to guard against data loss due to hardware failures, accidental deletions, or security breaches. Hetzner allows you to take storagebox as well as server snapshots so as to protect your data in the events of corruption or hardware failures. Their storage box has multiple RAID configurations which can withhold multiple disk failures.

6. **Monitoring and Intrusion Detection**: Set up monitoring tools to keep an eye on server performance and security. Implement intrusion detection systems (IDS) to detect and respond to unusual or malicious activities promptly. I use grafana, prometheus, node exporter storing nginx, mariadb, docker & node exporter logs.

7. **User Permissions and Access Control**: Configure user permissions and access control within PhotoPrism and your server (disable port 22 external access) to restrict who can view, modify, or delete photos and data. Only grant access to trusted individuals.

By implementing these security measures, you can rest assured that your photo collection is protected from threats and that your memories remain private. Building a secure foundation for your PhotoPrism setup ensures that you can enjoy your visual treasures without worry, knowing that they are safeguarded in the digital realm.

## Exploring the Limits of Hosting: The 4K Streaming Challenge

In my quest for the perfect media hosting solution, I embarked on an adventure with my trusty Raspberry Pi 4B. Raspberry Pi is a marvel of technology. Its compact size, energy efficiency, and versatility have made it a favorite for countless hobbyists and tinkerers worldwide. It's a budget-friendly solution with a vast community of supporters, allowing you to create projects limited only by your imagination.

The dream was to stream 4K videos in all their glory. However, as I fine-tuned my Raspberry Pi-based media server, I found myself facing below unexpected adversaries: 
* **ISP Upload Speeds: The Achilles Heel**: Despite investing in a high-speed plan, the challenge of streaming 4K videos remained elusive due to bandwidth limitations. While my Raspberry Pi proved more than capable, it was clear that a change in the ISP landscape was needed to realize the dream of seamless 4K streaming. Stay tuned for updates on my ongoing quest to conquer this streaming frontier.
* **FFMPEG transcodes**: Photoprism transcodes videos in a format modern browsers can play. For high quality video transcoding a GPU is highly recommended. RaspberryPi took ages to transcode large videos, often touching 70°C. You can have a cooling fan but on a long run it's not good for the health of the tiny Pi.

## Conclusion: Your PhotoPrism Journey Begins Here
In a world where photos hold the power to transport us back in time, elicit emotions, and tell our stories, having a well-organized and secure repository is invaluable. Our setup not only streamlines the process of cataloging and searching through your photos but also enhances the security and accessibility of your digital treasure trove.

So, what are you waiting for? Dive into this exciting journey of photo management and preservation. Build your own PhotoPrism setup, and let your visual memories shine brighter than ever before. The moments captured in your photos are meant to be relived, and with your new setup, they're just a click away. Your PhotoPrism journey begins here, and we can't wait to see where it takes you.