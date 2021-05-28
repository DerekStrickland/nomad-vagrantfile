# -*- mode: ruby -*-
# vi: set ft=ruby :

$nomad_server = <<SCRIPT
echo "param 1 $1"
echo "param 2 $2"
echo "param 3 $3"
echo "Installing Docker..."
sudo apt-get update
sudo apt-get remove docker docker-engine docker.io
echo '* libraries/restart-without-asking boolean true' | sudo debconf-set-selections
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common -y
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg |  sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) \
      stable"
sudo apt-get update
sudo apt-get install -y docker-ce
# Restart docker to make sure we get the latest version of the daemon if there is an upgrade
sudo service docker restart
# Make sure we can actually use docker as the vagrant user
sudo usermod -aG docker vagrant
sudo docker --version

# Packages required for nomad & consul
sudo apt-get install unzip curl vim -y

echo "Installing Nomad..."
NOMAD_VERSION=1.1.0
cd /tmp/
curl -sSL https://releases.hashicorp.com/nomad/${NOMAD_VERSION}/nomad_${NOMAD_VERSION}_linux_amd64.zip -o nomad.zip
unzip nomad.zip
sudo install nomad /usr/local/bin/nomad
sudo mkdir -p /etc/nomad.d
sudo chmod a+w /etc/nomad.d
sudo mkdir -p /opt/nomad
sudo chmod a+w /opt/nomad
(
cat <<-EOF
  [Unit]
  Description=Nomad
  Documentation=https://www.nomadproject.io/docs
  Wants=network-online.target
  After=network-online.target
  StartLimitBurst=3
  StartLimitIntervalSec=10

  [Service]
  ExecReload=/bin/kill -HUP $MAINPID
  ExecStart=/usr/local/bin/nomad agent -config /etc/nomad.d >> /opt/nomad/nomad.log
  KillMode=process
  KillSignal=SIGINT
  LimitNOFILE=infinity
  LimitNPROC=infinity
  Restart=on-failure
  RestartSec=20
  TasksMax=infinity

  [Install]
  WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/nomad.service
(
cat <<-EOF
  datacenter = "dc1"
  data_dir = "/opt/nomad"

  advertise {
    # Defaults to the first private IP address.
    http = "$3"
    rpc  = "$3"
    serf = "$3" # non-default ports may be specified
  }
EOF
) | sudo tee /etc/nomad.d/nomad.hcl
(
cat <<-EOF
  server {
    enabled = true
    bootstrap_expect = 3
    raft_protocol = 3
    server_join {
      retry_join = [ "$1", "$2" ]
      retry_max = 12
      retry_interval = "1m"
    }
  }
  autopilot {
    cleanup_dead_servers      = true
    last_contact_threshold    = "200ms"
    max_trailing_logs         = 250
    server_stabilization_time = "10s"
    enable_redundancy_zones   = false
    disable_upgrade_migration = false
    enable_custom_upgrades    = false
  }
EOF
) | sudo tee /etc/nomad.d/server.hcl
sudo systemctl enable nomad.service

echo "Installing Consul..."
CONSUL_VERSION=1.9.5
curl -sSL https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip > consul.zip
unzip /tmp/consul.zip
sudo install consul /usr/local/bin/consul
(
cat <<-EOF
  [Unit]
  Description=consul agent
  Requires=network-online.target
  After=network-online.target

  [Service]
  Restart=on-failure
  ExecStart=/local/bin/consul agent -dev
  ExecReload=/bin/kill -HUP $MAINPID

  [Install]
  WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/consul.service
sudo systemctl enable consul.service

for bin in cfssl cfssl-certinfo cfssljson
do
  echo "Installing $bin..."
  curl -sSL https://pkg.cfssl.org/R1.2/${bin}_linux-amd64 > /tmp/${bin}
  sudo install /tmp/${bin} /usr/local/bin/${bin}
done
nomad -autocomplete-install

sudo systemctl start consul
sudo systemctl start nomad

SCRIPT

Vagrant.configure(2) do |config|
  (1..3).each do |i|

    config.vm.box = "bento/ubuntu-18.04" # 18.04 LTS

    if i == 1
      config.vm.define "ns1" do |ns1|
        ns1.vm.hostname = "ns1"
        ns1.vm.network "private_network", ip: "192.168.0.1"
        ns1.vm.network "forwarded_port", guest: "4646", host: "4641"
        ns1.vm.provision "shell", inline: $nomad_server, privileged: false, args: "192.168.0.2 192.168.0.3 192.168.0.1"
      end
    elsif i == 2
      config.vm.define "ns2" do |ns2|
        ns2.vm.hostname = "ns2"
        ns2.vm.network "private_network", ip: "192.168.0.2"
        ns2.vm.network "forwarded_port", guest: "4646", host: "4642"
        ns2.vm.provision "shell", inline: $nomad_server, privileged: false, args: "192.168.0.1 192.168.0.3 192.168.0.2"
      end
    elsif i == 3
      config.vm.define "ns3" do |ns3|
        ns3.vm.hostname = "ns3"
        ns3.vm.network "private_network", ip: "192.168.0.3"
        ns3.vm.network "forwarded_port", guest: "4646", host: "4643"
        ns3.vm.provision "shell", inline: $nomad_server, privileged: false, args: "192.168.0.1 192.168.0.2 192.168.0.3"
      end
    end

    # Expose the nomad api and ui to the host
    # config.vm.network "forwarded_port", guest: 4646, host: 4646, auto_correct: true, host_ip: "127.0.0.1"
    # Increase memory for Parallels Desktop
    config.vm.provider "parallels" do |p, o|
      p.memory = "1024"
    end

    # Increase memory for Virtualbox
    config.vm.provider "virtualbox" do |vb|
          vb.memory = "1024"
    end

    # Increase memory for VMware
    ["vmware_fusion", "vmware_workstation"].each do |p|
      config.vm.provider p do |v|
        v.vmx["memsize"] = "1024"
      end
    end
  end
end
