# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# 노트북 랩 인프라 - Vagrant 정의
# 4대 VM(dns01/web01/db01/client01)을 Host-only(192.168.56.0/24) +
# NAT 이중 네트워크로 구성한다.
#
# - VMware의 기본 공유 폴더(/vagrant)가 자동으로 마운트되지 않아,
#   ansible/ 디렉토리를 file provisioner로 SSH를 통해 직접 복사한다.
# - ansible_local의 기본 pip 설치는 저사양 VM에서 컴파일 시간이 길어져
#   SSH 세션이 끊기므로, dnf(ansible-core, AppStream 기본 포함)로 먼저
#   설치해두고 ansible_local에는 install_mode "default"를 사용한다.
# - 각 VM은 자기 자신만 인벤토리에 포함시켜(다른 VM을 찾으려 하지 않음)
#   site.yml 중 자신의 role에 해당하는 play만 실행한다.
#
# 사전 준비 (Windows PowerShell, 관리자 권한 불필요):
#   vagrant plugin install vagrant-vmware-desktop
#   vagrant box add generic/rocky9 --provider=vmware_desktop

VMS = {
  "dns01"    => { ip: "192.168.56.11", mem: 1024, cpu: 1, role: "dns" },
  "web01"    => { ip: "192.168.56.12", mem: 1024, cpu: 1, role: "web" },
  "db01"     => { ip: "192.168.56.13", mem: 1536, cpu: 1, role: "db" },
  "client01" => { ip: "192.168.56.14", mem: 1024, cpu: 1, role: "client" },
  "monitor01" => { ip: "192.168.56.15", mem: 768, cpu: 1, role: "monitor" },
}

Vagrant.configure("2") do |config|
  config.vm.box = "generic/rocky9"

  VMS.each do |name, opts|
    config.vm.define name do |node|
      node.vm.hostname = "#{name}.lab.local"

      # Host-only 내부망 (고정 IP)
      node.vm.network "private_network", ip: opts[:ip]

      node.vm.provider "vmware_desktop" do |v|
        v.memory = opts[:mem]
        v.cpus = opts[:cpu]
      end

      # dnf로 ansible-core를 먼저 설치 (pip 컴파일보다 훨씬 빠르고 안정적)
      node.vm.provision "shell", inline: <<-SHELL
        for i in $(seq 1 15); do
          dnf install -y ansible-core && break
          echo "네트워크/DNS 준비 대기 중... ($i/15)"
          sleep 2
        done
      SHELL

      # /vagrant 공유폴더에 의존하지 않고, ansible/ 디렉토리를 SSH로 직접 업로드
      node.vm.provision "file", source: "ansible", destination: "/home/vagrant/ansible"

      # Ansible Galaxy 컬렉션 설치 (업로드된 경로 기준)
      node.vm.provision "shell", privileged: false, inline: <<-SHELL
        ansible-galaxy collection install -r /home/vagrant/ansible/requirements.yml
      SHELL

      node.vm.provision "ansible_local" do |ansible|
        ansible.provisioning_path = "/home/vagrant/ansible"
        ansible.playbook = "site.yml"
        ansible.install_mode = "default"
        ansible.limit = "all"
        # 이 VM 자신만 인벤토리에 등록. lab 그룹 + 자신의 role 그룹에만 소속시켜서
        # site.yml에서 자신에게 해당하는 play만 실행되게 한다.
        ansible.groups = {
          "lab"        => [name],
          opts[:role]  => [name],
        }
      end
    end
  end
end
