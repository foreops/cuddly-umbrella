WORKER_PREFIX="worker"
NUM_WORKERS=2
PRIVATE_NET="172.28.128"
MAIN_NODE="main-node"
MAIN_NODE_IP=PRIVATE_NET+".10"

# Download from https://spark.apache.org/downloads.html and locate with the Vagrantfile
SPARK_ARCHIVE="spark-3.0.0-bin-hadoop2.7.tgz"

#global script
$global = <<SCRIPT
#check for private key for vm-vm comm
[ -f /vagrant/id_rsa ] || {
  ssh-keygen -t rsa -f /vagrant/id_rsa -q -N ''
}
#deploy key
[ -f /home/vagrant/.ssh/id_rsa ] || {
    cp /vagrant/id_rsa /home/vagrant/.ssh/id_rsa
    chmod 0600 /home/vagrant/.ssh/id_rsa
}
#allow ssh passwordless
grep 'vagrant@node' ~/.ssh/authorized_keys &>/dev/null || {
  cat /vagrant/id_rsa.pub >> ~/.ssh/authorized_keys
  chmod 0600 ~/.ssh/authorized_keys
}
#exclude workers from host checking
cat > ~/.ssh/config <<EOF
Host #{WORKER_PREFIX}*
   StrictHostKeyChecking no
   UserKnownHostsFile=/dev/null
EOF
#populate /etc/hosts
echo #{MAIN_NODE_IP} #{MAIN_NODE} | sudo tee -a /etc/hosts &>/dev/null
for x in {21..#{20+NUM_WORKERS}}; do
  grep #{PRIVATE_NET}.${x} /etc/hosts &>/dev/null || {
      echo #{PRIVATE_NET}.${x} #{WORKER_PREFIX}${x##?} | sudo tee -a /etc/hosts &>/dev/null
  }
done
#end script
SCRIPT

Vagrant.configure("2") do |config|
  
  config.vm.box = "ubuntu/xenial64"
  config.vm.provision "shell", privileged: false, inline: $global

  #worker nodes
  (1..NUM_WORKERS).each do |i|
    vm_name = "#{WORKER_PREFIX}#{i}"
    config.vm.define vm_name do |node|
      node.vm.hostname = vm_name
      ip="#{PRIVATE_NET}.#{20+i}"
      node.vm.network "private_network", ip: ip
      
      node.vm.provider "virtualbox" do |vb|
        vb.name = vm_name
        vb.gui = false
        vb.memory = 4096
        vb.cpus = 2
      end
      node.vm.provision "shell", privileged: false, inline: <<-SHELL
        sudo add-apt-repository ppa:deadsnakes/ppa -y
        sudo apt-get update -y
        sudo apt-get install -y python3.8 python3.8-dev ntp avahi-daemon default-jdk git
        mkdir /home/vagrant/spark
        tar xvf /vagrant/#{SPARK_ARCHIVE} -C /home/vagrant/spark --strip 1
        echo export SPARK_HOME=/home/vagrant/spark >> /home/vagrant/.bashrc
        echo export PATH=$PATH:/home/vagrant/spark/bin::/home/vagrant/spark/sbin >> /home/vagrant/.bashrc
        echo export PYSPARK_PYTHON=/usr/bin/python3.8 >> /home/vagrant/.bashrc
        echo SPARK_LOCAL_IP=#{ip} >> /home/vagrant/spark/conf/spark-env.sh
        echo SPARK_MASTER_HOST=#{MAIN_NODE_IP} >> /home/vagrant/spark/conf/spark-env.sh
      SHELL

    end
  end

  # main node:
  config.vm.define "#{MAIN_NODE}" do |node|
    node.vm.hostname = "#{MAIN_NODE}"
    node.vm.network "private_network", ip: MAIN_NODE_IP

    node.vm.provider "virtualbox" do |vb|
      vb.name = "#{MAIN_NODE}"
      vb.gui = false
      vb.memory = 2048
      vb.cpus = 2
    end

    node.vm.provision "shell", privileged: false, inline: <<-SHELL 
      sudo add-apt-repository ppa:deadsnakes/ppa -y
      sudo apt-get update -y
      sudo apt-get install -y python3.8 python3.8-dev ntp default-jdk git
      mkdir /home/vagrant/spark
      tar xvf /vagrant/#{SPARK_ARCHIVE} -C /home/vagrant/spark --strip 1
      echo export SPARK_HOME=/home/vagrant/spark >> /home/vagrant/.bashrc
      echo export PATH=$PATH:/home/vagrant/spark/bin::/home/vagrant/spark/sbin >> /home/vagrant/.bashrc
      echo export PYSPARK_PYTHON=/usr/bin/python3.8 >> /home/vagrant/.bashrc
      echo export SPARK_MASTER_HOST=#{MAIN_NODE_IP} >> /home/vagrant/.bashrc
      echo spark.master spark://#{MAIN_NODE_IP}:7077 >> /home/vagrant/spark/conf/spark-defaults.conf
      echo SPARK_LOCAL_IP=#{MAIN_NODE_IP} >> /home/vagrant/spark/conf/spark-env.sh
      echo SPARK_MASTER_HOST=#{MAIN_NODE_IP} >> /home/vagrant/spark/conf/spark-env.sh
      echo #{MAIN_NODE} | tee -a /home/vagrant/spark/conf/slaves &>/dev/null
      for x in {1..#{NUM_WORKERS}}; do
          echo #{WORKER_PREFIX}${x} | tee -a /home/vagrant/spark/conf/slaves &>/dev/null
      done
      sudo cp /vagrant/spark-init.service /etc/systemd/system
      sudo systemctl daemon-reload
      sudo systemctl enable spark-init.service
    SHELL

    node.vm.provision "shell", run: "always", privileged: true, inline: "systemctl start spark-init.service"
  end
end