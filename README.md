# checkmk
Step-by-step guide to setting up a **Checkmk server monitoring system** on a local PC using **Vagrant** with **Ansible**, and monitoring three agent servers (**2 Ubuntu, 1 Windows Server**).  

---

## **1. Setup Checkmk Server with Vagrant and Ansible**  

### **1.1 Install Prerequisites**  
Ensure you have the following installed:  
- **Vagrant** (for VM provisioning)  
- **VirtualBox** or **Libvirt** (for running VMs)  
- **Ansible** (for automation)  
- **Checkmk** (Raw Edition or Cloud Edition)

### **1.2 Create Vagrantfile for Checkmk Server**  

Create a working directory and a `Vagrantfile`:  

```bash
mkdir ~/checkmk-monitoring && cd ~/checkmk-monitoring
touch Vagrantfile
```

Edit the `Vagrantfile`:  

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.hostname = "checkmk-server"
  config.vm.network "private_network", type: "dhcp"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "checkmk_install.yml"
  end
end
```

### **1.3 Create Ansible Playbook to Install Checkmk Server**  

Create `checkmk_install.yml`:

```yaml
- name: Install Checkmk Monitoring Server
  hosts: all
  become: yes
  tasks:
    - name: Update apt packages
      apt:
        update_cache: yes

    - name: Install dependencies
      apt:
        name:
          - wget
          - curl
          - apt-transport-https
          - gnupg
        state: present

    - name: Download and Install Checkmk
      shell: |
        wget https://download.checkmk.com/checkmk/2.1.0p23/check-mk-raw-2.1.0p23_0.focal_amd64.deb
        dpkg -i check-mk-raw-2.1.0p23_0.focal_amd64.deb

    - name: Create Checkmk Site
      shell: |
        omd create monitoring
        systemctl enable omd@monitoring
        systemctl start omd@monitoring

    - name: Print Checkmk Admin Credentials
      shell: |
        omd config monitoring
        omd status monitoring
```

### **1.4 Start Checkmk Server**  

```bash
vagrant up
```

Once up, access Checkmk:  
👉 Open **http://<Checkmk-Server-IP>/monitoring**  

---

## **2. Secure Checkmk Server with Local TLS Certificate**  

### **2.1 Generate TLS Certificates (Self-Signed)**  

On the Checkmk server:  

```bash
mkdir -p /etc/ssl/checkmk
cd /etc/ssl/checkmk

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout checkmk.key -out checkmk.crt \
  -subj "/C=US/ST=YourState/L=YourCity/O=YourOrg/CN=checkmk.local"

chmod 600 checkmk.*
```

### **2.2 Configure Apache for TLS in Checkmk**  

Edit the Checkmk site Apache config:  

```bash
nano /omd/sites/monitoring/etc/apache/apache.conf
```

Add:

```apache
<VirtualHost *:443>
    ServerAdmin admin@checkmk.local
    DocumentRoot /omd/sites/monitoring/var/www
    ServerName checkmk.local

    SSLEngine on
    SSLCertificateFile /etc/ssl/checkmk/checkmk.crt
    SSLCertificateKeyFile /etc/ssl/checkmk/checkmk.key

    <Directory "/omd/sites/monitoring/var/www">
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Enable SSL and restart Apache:  

```bash
a2enmod ssl
systemctl restart apache2
```

Now access **https://checkmk.local/monitoring** securely.

---

## **3. Setup Agents for 3 Servers (2 Ubuntu, 1 Windows)**  

### **3.1 Create Vagrantfile for Agent VMs**  

Modify `Vagrantfile` to add agent servers:

```ruby
Vagrant.configure("2") do |config|
  # Checkmk Server
  config.vm.define "checkmk-server" do |server|
    server.vm.box = "ubuntu/focal64"
    server.vm.hostname = "checkmk-server"
    server.vm.network "private_network", type: "dhcp"
    server.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = 2
    end
    server.vm.provision "ansible" do |ansible|
      ansible.playbook = "checkmk_install.yml"
    end
  end

  # Ubuntu Agent 1
  config.vm.define "ubuntu-agent-1" do |agent|
    agent.vm.box = "ubuntu/focal64"
    agent.vm.hostname = "ubuntu-agent-1"
    agent.vm.network "private_network", type: "dhcp"
    agent.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end
    agent.vm.provision "ansible" do |ansible|
      ansible.playbook = "agent_install.yml"
    end
  end

  # Ubuntu Agent 2
  config.vm.define "ubuntu-agent-2" do |agent|
    agent.vm.box = "ubuntu/focal64"
    agent.vm.hostname = "ubuntu-agent-2"
    agent.vm.network "private_network", type: "dhcp"
    agent.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end
    agent.vm.provision "ansible" do |ansible|
      ansible.playbook = "agent_install.yml"
    end
  end

  # Windows Agent
  config.vm.define "windows-agent" do |agent|
    agent.vm.box = "gusztavvargadr/windows-server"
    agent.vm.hostname = "windows-agent"
    agent.vm.network "private_network", type: "dhcp"
    agent.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = 2
    end
  end
end
```

---

## **4. Install Checkmk Agents**  

### **4.1 Ansible Playbook for Ubuntu Agents**  

Create `agent_install.yml`:

```yaml
- name: Install Checkmk Agent
  hosts: all
  become: yes
  tasks:
    - name: Download and Install Checkmk Agent
      shell: |
        wget https://download.checkmk.com/checkmk/2.1.0p23/check-mk-agent_2.1.0p23-1_all.deb
        dpkg -i check-mk-agent_2.1.0p23-1_all.deb

    - name: Enable Agent xinetd Service
      systemd:
        name: xinetd
        enabled: yes
        state: started
```

Run the setup:

```bash
vagrant up
```

---

## **5. Monitor Servers in Checkmk**  

1. **Login to Checkmk Web UI**  
2. Go to **Setup > Agents**  
3. Add the agent hosts (`ubuntu-agent-1`, `ubuntu-agent-2`, `windows-agent`)  
4. Run a discovery scan  
5. Enable monitoring  

---


