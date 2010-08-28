#
# Chef-Solo Capistrano Bootstrap
#
# usage:
#   cap chef:bootstrap <role> <remote_host>
#
# NOTICE OF LICENSE
#
# Copyright (c) 2010 Mike Smullin <mike@smullindesign.com>
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.


# configuration
set :scm, :git
set :ssh_options, { :forward_agent => true, :port => 22 }
default_run_options[:pty] = true # fix to display interactive password prompts
# abort "please specify target server as last argument" unless ARGV.count >= 2
role :target, ARGV[-1]
cwd = File.expand_path(File.dirname(__FILE__))

# tasks
namespace :chef do
  desc "Bootstrap an Ubuntu 10.04 server and kick-start Chef-Solo"
  task :bootstrap, roles: :target do
    install_rvm_ruby
    #install_ree
    install_chef
    install_cookbook_repo
    install_dna
    solo

    exit # subsequent args are not tasks to be run
  end

  desc "Install Ruby Enterprise Edition"
  # see also: http://www.rubyenterpriseedition.com/download.html
  task :install_ree, roles: :target do
    ree_version = 'ruby-enterprise-1.8.7-2010.02'
    ree_prefix = '/usr/local'
    sudo 'aptitude install -y zlib1g-dev libssl-dev libreadline5-dev' # REE dependencies
    run "cd /tmp && wget http://rubyforge.org/frs/download.php/71096/#{ree_version}.tar.gz && tar zxvf #{ree_version}.tar.gz"
    sudo "mkdir -p #{ree_prefix}/lib/ruby/gems/1.8/gems" # workaround for bug in REE installer
    sudo "/tmp/#{ree_version}/installer -a #{ree_prefix} --dont-install-useful-gems --no-dev-docs"
    run "echo 'gem: --no-ri --no-rdoc' | #{sudo} tee -a /etc/gemrc"
  end

  desc "install Ruby (using RVM)"
  # see also: http://rohitarondekar.com/articles/installing-rails3-beta3-on-ubuntu-using-rvm
  task :install_rvm_ruby, roles: :target do
    rvm_ruby_version = '1.9.2-p0'
    set :default_environment, {
      'PATH'         => "$HOME/.rvm/bin:$PATH",
      'RUBY_VERSION' => "ruby #{rvm_ruby_version}",
      'GEM_HOME'     => "$HOME/.rvm/gems/ruby-#{rvm_ruby_version}",
      'GEM_PATH'     => "$HOME/.rvm/gems/ruby-#{rvm_ruby_version}:$HOME/.rvm/gems/ruby-#{rvm_ruby_version}@global",
      'BUNDLE_PATH'  => "$HOME/.rvm/gems/ruby-#{rvm_ruby_version}"
    }
    sudo 'aptitude install -y curl git-core'
    run 'mkdir -p ~/.rvm/src/ && cd ~/.rvm/src && rm -rf ./rvm/ && git clone --depth 1 git://github.com/wayneeseguin/rvm.git && cd rvm && ./install'
    run %q(sed -i 's/^\[/# [/' ~/.bashrc)
    run %q(echo '[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm"' >> ~/.bashrc)

    # dependencies for compiling Ruby
    sudo 'aptitude install -y bison build-essential autoconf zlib1g-dev libssl-dev libxml2-dev libreadline6-dev'
    # dependencies for compiling our RubyGems gems
    sudo 'aptitude install -y libpq-dev libmysql-ruby libmysqlclient-dev sqlite3 libsqlite3-dev libxslt-dev libxml2-dev imagemagick'

    # openssl package for RVM
    run 'rvm package install openssl'

    # readline package for RVM
    run 'rvm package install readline'

    # installing ruby
    run "rvm install #{rvm_ruby_version} -C --with-openssl-dir=$rvm_path/usr,--with-readline-dir=$rvm_path/usr"
    run "rvm use #{rvm_ruby_version} --default"
    run "echo 'gem: --no-ri --no-rdoc' | #{sudo} tee /etc/gemrc"
  end

  desc "Install Chef and Ohai gems as root"
  task :install_chef, roles: :target do
    # install gems
    sudo 'gem source -a http://gems.opscode.com/'
    sudo 'gem install ohai chef'
  end

  desc "Install Cookbook Repository from cwd"
  task :install_cookbook_repo, roles: :target do
    cookbook_dir = '/var/chef-solo'
    sudo "mkdir -m 0775 -p #{cookbook_dir}"
    sudo "chown `whoami`.`whoami` #{cookbook_dir}"
    rsync cwd + '/', cookbook_dir
    sudo 'mkdir -p /etc/chef'
    sudo_put %Q(
file_cache_path "#{cookbook_dir}"
cookbook_path ["#{cookbook_dir}/cookbooks", "#{cookbook_dir}/site-cookbooks"]
role_path "#{cookbook_dir}/roles"
), '/etc/chef/solo.rb',
      via: :scp,
      mode: '0644'
  end

  desc "Install ./dna/*.json for specified node"
  task :install_dna, roles: :target do
    node = ARGV[-2]
    sudo 'mkdir -p /etc/chef'
    sudo_put File.read("#{cwd}/dna/#{node}.json"), '/etc/chef/dna.json',
      via: :scp,
      mode: '0644'
  end

  desc "Execute Chef-Solo"
  task :solo, roles: :target do
    sudo 'chef-solo -c /etc/chef/solo.rb -j /etc/chef/dna.json -l debug'
  end

  desc "Remove all traces of Chef"
  task :cleanup, roles: :target do
    sudo 'rm -rf /etc/chef /var/chef-solo'
    sudo 'gem uninstall -ay chef ohai'
  end
end


# helpers
def sudo_put(data, target, opts={})
  tmp = "/tmp/~tmp-#{rand(9999999)}"
  put data, tmp, opts
  on_rollback { run "rm #{tmp}" }
  sudo "cp -f #{tmp} #{target} && rm #{tmp}"
end

def sudo_upload_tar(from, to, opts={})
  tmp = "tmp-#{rand(9999999)}.tar.gz"
  system %Q(tar --exclude-vcs --exclude .deb -czf /tmp/"#{tmp}" "#{from}")
  put(File.read("/tmp/#{tmp}"), "#{to}/#{tmp}", opts)
  sudo <<-CMD
    tar -C #{to} zxvf #{to}/#{tmp} &&
    rm -rf #{to}/#{tmp}
  CMD
  system("rm /tmp/#{tmp}")
  on_rollback { run "rm /tmp/#{tmp}" }
end

def rsync(from, to)
  sudo 'aptitude install -y rsync'
  find_servers_for_task(current_task).each do |server|
    puts `rsync -avz -e ssh "#{from}" "#{ENV['USER']}@#{server}:#{to}" \
      --exclude ".svn" --exclude ".git"`
  end
end