# ELK Homelab Setup

A comprehensive guide for setting up an Elasticsearch, Kibana, and Fleet Server (ELK Stack) environment on Ubuntu 24 using Docker Compose, plus Zeek network monitoring on Raspberry Pi.

## 📚 Documentation

- **[ELK Fleet Setup Guide](ELK-Fleet-Setup-8.17.4.md)** - Complete ELK Stack setup with Fleet Server
- **[Zeek Integration Guide](zeek.md)** - Zeek network monitoring with Elastic integration

## 🎯 What's Included

### ELK Stack Setup
- Set up Elasticsearch 8.17.4 in a Docker container
- Configure Kibana with proper service account authentication
- Add Fleet Server for centralized agent management
- Install and enroll Elastic Agents
- Troubleshoot common issues and errors

### Zeek Network Monitoring
- Install Zeek on Raspberry Pi 4 using Docker
- Configure port mirroring for network traffic analysis
- Integrate Zeek logs with Elastic Stack via Fleet
- JSON log format configuration
- Common troubleshooting scenarios

## ⚙️ Key Features

- **Docker Compose Setup**: Easy deployment and management
- **Service Account Tokens**: Proper authentication without using the elastic superuser
- **Fleet Integration**: Centralized agent management and monitoring
- **Troubleshooting Guide**: Solutions to common problems
- **Production Ready**: Security considerations and best practices

## 🚀 Quick Start

### ELK Stack Setup

1. **Prerequisites**
   - Ubuntu Server 24.04
   - Docker and Docker Compose
   - Minimum 8 GB RAM
   - Ports: 9200, 5601, 8220

2. **Setup Steps**
   ```bash
   cd ~
   mkdir elk && cd elk
   ```

3. **Follow the guide**
   - Read the [ELK Fleet Setup Guide](ELK-Fleet-Setup-8.17.4.md)
   - Set up your `.env` file
   - Create your `docker-compose.yml`
   - Follow the step-by-step instructions

### Zeek Network Monitoring

1. **Prerequisites**
   - Raspberry Pi 4 with 64-bit OS
   - Docker installed
   - Managed switch with port mirroring capability
   - Elastic Agent enrolled in Fleet

2. **Setup Steps**
   - Read the [Zeek Integration Guide](zeek.md)
   - Configure port mirroring on your switch
   - Deploy Zeek container
   - Configure Fleet integration

## 📋 What You'll Learn

### ELK Stack
- How to properly configure Elasticsearch security
- Service account token generation and management
- Kibana connection best practices
- Fleet Server setup and configuration
- Elastic Agent enrollment and management
- Common pitfalls and how to avoid them

### Zeek Network Monitoring
- Port mirroring configuration on managed switches
- Zeek deployment in Docker containers
- JSON log format configuration for Elastic integration
- Fleet integration for Zeek logs
- Network traffic analysis and troubleshooting

## 🔧 System Requirements

### ELK Stack Server
- **Operating System**: Ubuntu Server 24.04
- **Memory**: 8 GB RAM (minimum)
- **Storage**: Adequate disk space for Elasticsearch data
- **Network**: Open ports 9200, 5601, and 8220

### Zeek Sensor (Raspberry Pi)
- **Hardware**: Raspberry Pi 4
- **Operating System**: 64-bit OS (Raspberry Pi OS or Ubuntu)
- **Docker**: Installed and configured
- **Network**: Managed switch with port mirroring support

## 📖 Topics Covered

### ELK Stack Guide
1. System configuration (vm.max_map_count)
2. Docker Compose setup
3. Elasticsearch configuration
4. Kibana with service account tokens
5. Fleet Server deployment
6. Elastic Agent installation (package and container)
7. Troubleshooting common issues
8. Security considerations

### Zeek Integration Guide
1. Port mirroring setup on managed switches
2. Zeek Docker deployment
3. JSON log format configuration
4. Fleet integration for network logs
5. Log verification in Elasticsearch
6. Common issues and solutions
7. Alternative Filebeat module setup

## 🐛 Troubleshooting

Both guides include detailed troubleshooting sections for:

### ELK Stack
- Container health check failures
- Authentication errors
- Service account token issues
- Fleet enrollment problems
- Network and firewall configuration

### Zeek Integration
- JSON parsing errors in pipeline
- Container restart loops
- Missing log files
- Port mirroring configuration
- Network interface detection issues

## 🔐 Security Notes

While this guide focuses on getting a working setup quickly, it includes a section on security considerations for production deployments:

- TLS/SSL configuration
- Encryption key management
- Proper token rotation
- Firewall configuration

## 📝 Version

Current version: **8.17.4**

This guide is specifically written for Elasticsearch 8.17.4, Kibana 8.17.4, and Elastic Agent 8.17.4.

## 🤝 Contributing

Feel free to open issues or submit pull requests if you find errors or have suggestions for improvements.

## 📄 License

This documentation is provided as-is for educational and homelab purposes.

## 🌟 Features

- **Comprehensive Documentation**: Step-by-step instructions with explanations
- **Tested Solutions**: All configurations tested in real homelab environments
- **Troubleshooting Focus**: Common issues and their solutions included
- **Network Monitoring**: Complete Zeek integration for traffic analysis
- **Docker-Based**: Easy deployment and management
- **Security Aware**: Best practices and security considerations included

## 🤝 Contributing

Contributions, issues, and feature requests are welcome! Feel free to check the issues page.

---

**Note**: This is a homelab setup guide. For production environments, additional security hardening and best practices should be implemented.
