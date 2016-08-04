require 'ipaddr'
require 'yaml'

$cwd = Pathname __dir__
config_file_env_var='VAGRANT_CONFIG_FILE'
config_file = $cwd.join 'vagrant-config.yml'

def parse_config(config_filename)
  begin
    config = YAML.load_file config_filename
  rescue Errno::ENOENT
    puts "No config file exists at #{config_filename}, will create…\n\n"
  rescue Exception => e
    warn <<ERROR

An exception occured while reading the config from a file, #{config_filename}.

The exception was:

#{e}

The requested Vagrant operation will still be performed, but default config info will be used.

ERROR
  ensure
    config ||= {shared_repos: []}
  end

  return symbolize_keys config
end

def symbolize_keys(hash)
  return hash.inject({}) do |result, (key, value)|
    new_key = key.to_sym
    new_value = case value
                when Hash then symbolize_keys(value)
                when Array then value.map! { |element| symbolize_keys element }
                else value
                end
    result[new_key] = new_value
    result
  end
end

def stringify_keys(hash)
  return hash.inject({}) do |result, (key, value)|
    new_key = key.to_s
    new_value = case value
                when Hash then symbolize_keys(value)
                when Array then value.map! { |element| stringify_keys element }
                else value
                end
    result[new_key] = new_value
    result
  end
end

def host_ip_address(guest_ip_address)
  begin
    nics = `VBoxManage list hostonlyifs`.split("\n\n").map do |net|
      Hash[
        # Separate the key/value pairs which are on each line
        net.split("\n").map do |line|
          # Chomp ":" off the end of all values
          line.split().each { |value| value.chomp!(":") }
        end
      ]
    end
  rescue Errno::ENOENT
    nics = {}
  end

  matching_nics = nics.select do |nic|
    IPAddr.new(nic['IPAddress']).mask(nic['NetworkMask']) == IPAddr.new(guest_ip_address).mask(nic['NetworkMask'])
  end

  if matching_nics.length > 1
    warn 'More than one matching host only interface, behaviour may be unpredictable.'
  elsif matching_nics.length == 0
    host_ip = IPAddr.new(guest_ip_address).mask('255.255.255.0').succ
    warn "No matching host only interface, will assume host is at #{host_ip}."
    return host_ip
  end

  return IPAddr.new matching_nics[0]['IPAddress']
end

def get_shared_repos_info(repos=[])
  # Array#reject creates a new array, but it contains the same object pointers as the original.
  # This means that we can loop over and make valid the invalid repos without touching
  # already valid ones, and they'll all still be in the shared_repos array.
  invalid_repos = repos.reject do |repo|
    repo.member?(:host_path) and
    repo.member?(:version) and
    repo.member?(:remote) and
    repo.member?(:guest_path)
  end

  if invalid_repos.length != 0
    puts "Please correct issues with these repos:\n\n#{invalid_repos}"
    exit 1
  end

  # TODO convert repo paths to absolute paths

  return repos
end


# Get the config from the file set in the env var or the default file
config_file = ENV[config_file_env_var] || config_file
config = parse_config config_file

# ARGV is a list of the arguments to this script
# `&` returns the intersection of two lists
# `%w(foo bar)` is `['foo', 'bar']`
# Order of operations applies `&` before `!=`
if ARGV & %w(up --provision provision) != []

  repos = config[:shared_repos]
  shared_repos = get_shared_repos_info repos

end

Vagrant.configure(2) do |vagrant|
  guest_ip_address = '192.168.33.29'
  hostname = ENV['VAGRANT_HOSTNAME'] || 'workshop.local.rf29.net'
  user_key_file = ENV['GITHUB_PRIVATE_KEY'] || '~/.ssh/id_rsa'

  vagrant.vm.box = 'ubuntu/trusty64'
  vagrant.vm.box_version = '20150817.0.0'

  vagrant.vm.network 'private_network', ip: guest_ip_address
  vagrant.vm.hostname = hostname

  vagrant.vm.provision :ansible do |ansible|
    ansible.sudo = true
    ansible.playbook = 'ansible/playbook.yml'
    extra_vars = {
        guest_ip_address: guest_ip_address,
        host_ip_address: host_ip_address(guest_ip_address),
        hostname: hostname,
        user_key_file: user_key_file,
        shared_repos: shared_repos,
    }
    if ENV.key? 'ANSIBLE_SUDO_PASS'
      extra_vars.store :ansible_sudo_pass, ENV['ANSIBLE_SUDO_PASS']
    else
      ansible.ask_sudo_pass = true
    end
    ansible.extra_vars = extra_vars
  end
end
