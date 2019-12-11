# -*- mode: ruby -*-
# vi: set ft=ruby :

# -------------------------------------------------------------
# Exit if no environment name specified.
# -------------------------------------------------------------

environments = [
    "dev",
    "stage",
    "prod"
]
commandsToCheck = [
    "destroy",
    "halt",
    "provision",
    "reload",
    "resume",
    "suspend",
    "up"
  ]


  if ENV['RUN_BY_PYTHON'] == 'yes'
    puts "***** Run using python vagrant tool *****"
    enteredCommand = ENV['VAGRANT_RUN_COMMAND']
    chosen_environment = ENV['VAGRANT_ENV_COMMAND']
  elsif
    enteredCommand = ARGV[0]
    if ARGV[1] == 'dev' || ARGV[2] == 'dev'
       chosen_environment = 'dev'
    elsif ARGV[1] == 'stage' || ARGV[2] == 'stage'
       chosen_environment = 'stage'
    elsif ARGV[1] == 'prod' || ARGV[2] == 'prod'
       chosen_environment = 'prod'
    else
       chosen_environment = 'None'
    end
  end



  # Is this one of the problem commands?
      if commandsToCheck.include?(enteredCommand)
        # Is this command lacking any other supported environments ? e.g. "vagrant destroy dev".
        if not (environments.include?chosen_environment)
          puts "You must use 'vagrant #{enteredCommand} #{environments.join("/")}....'"
          puts "Run 'vagrant status' to view VM names."
          exit 1
        end
    end



puts "Working on environment: #{chosen_environment}" if chosen_environment != 'None'

require 'yaml'
Vagrant.require_version ">= 2.2.5"


required_plugins = %w( vagrant-env vagrant-docker-compose vagrant-disksize)
required_plugins.each do |plugin|
    exec "vagrant plugin install #{plugin}" unless Vagrant.has_plugin? plugin || ARGV[0] == 'plugin'
end

if chosen_environment != 'dev' # aws plugin is needed only for non dev environment
    if Vagrant::Util::Platform.windows?
        # needed for windows as prerequisite for vagrant-aws
        required_plugins = [
        {"fog-ovirt" => {"version" => "1.0.1"}},
        "vagrant-aws",
        ]
    else
        required_plugins = [
            "vagrant-aws",
        ]
    end
end

puts "Start vagrant configure."

Vagrant.configure("2") do |config|
  config.vagrant.plugins = required_plugins

  if File.exist?('.env.local')
      Dotenv.load('.env.local')
  end

  config.env.enable

  config.vm.provision "shell",
    path: "scripts/bootstrap.sh",
    args: "#{chosen_environment}"\
          " #{ENV['BASE_FOLDER']}",
    run: "always"

  config.vm.provision :docker
  config.vm.provision :docker_compose,
    compose_version: "1.24.1"
  config.vm.provision "docker compose provision",
    type: "shell",
    path: "scripts/docker-compose-provision.sh",
    args: "#{chosen_environment}"\
          " #{ENV['BASE_FOLDER']}"\
          " #{ENV['WEB_HOST_PORT']}"\
          " #{ENV['WEB_GUEST_PORT']}"\
          " #{ENV['WEB_HOST_SECURE_PORT']}"\
          " #{ENV['WEB_GUEST_SECURE_PORT']}",
    run: "always"

  config.trigger.after :up do |trigger|
    trigger.info = "Loading database"
    trigger.run_remote = {
      path: "scripts/load-db.sh",
      args: "#{chosen_environment} #{ENV['BASE_FOLDER']}"
    }
    trigger.on_error = :continue
  end
  config.trigger.before :destroy do |trigger|
    trigger.info = "Dumping database"
    trigger.run_remote = {
      path: "scripts/dump-db.sh",
      args: "#{chosen_environment} #{ENV['BASE_FOLDER']}"
    }
    trigger.on_error = :continue
  end

  if chosen_environment == "dev"
    config.vm.define "dev" do |dev|

      puts "Inside config.vm.define Dev Vagrant file"
      if File.exist?('.env.local')
        Dotenv.load('.env.local')
      end

      config.env.enable
      dev.vm.network "forwarded_port",
              guest: ENV['DEV_WEB_GUEST_PORT'],
              host:  ENV['DEV_WEB_HOST_PORT']
      dev.vm.network "forwarded_port",
              guest: ENV['DEV_WEB_GUEST_SECURE_PORT'],
              host:  ENV['DEV_WEB_HOST_SECURE_PORT']
      dev.vm.network "forwarded_port",
              guest: ENV['VAULT_GUEST_PORT'],
              host:  ENV['VAULT_HOST_PORT']
      dev.vm.network "private_network", ip: "10.0.0.1"

      dev.vm.synced_folder '.', ENV['BASE_FOLDER'],
        disabled: false,
        type: "rsync",
        rsync__exclude: ['.git/', 'workshops/', 'venv/']

      dev.vm.synced_folder ENV['DEV_VAULT_FOLDER'], '/vault',
        owner: 100,
        group: 1000

      dev.disksize.size = '10GB'

      dev.vm.provider :virtualbox do |virtualbox,override|
        dev.vm.box = ENV['DEV_VAGRANT_BOX']
  #     dev.vm.box_url = "https://app.vagrantup.com/ubuntu/boxes/bionic64"
        virtualbox.memory = 1024
        virtualbox.cpus = 2
      end

      dev.vm.provider :aws do |aws,override|
        aws.keypair_name = ENV['STAGE_KEYPAIR_NAME']
        aws.ami = ENV['STAGE_AMI']
        aws.instance_type = ENV['STAGE_INSTANCE_TYPE']
        aws.region = ENV['STAGE_REGION']
        aws.subnet_id = ENV['STAGE_SUBNET_ID']
        aws.security_groups = ENV['STAGE_SECURITY_GROUPS']
        aws.associate_public_ip = true
        aws.iam_instance_profile_name = ENV['STAGE_INSTANCE_PROFILE_NAME']
        aws.aws_dir = ENV['HOME'] + "/.aws/"
        aws.aws_profile = "#{ENV['STAGE_AWS_PROFILE']}"

        override.ssh.username = "ubuntu"
  #			override.ssh.private_key_path = ENV['STAGE_SSH_PRIVATE_KEY_PATH']
      end


    end
  end

  if chosen_environment == "stage"
    config.vm.define "stage" do |stage|

      puts "Inside config.vm.define Stage Vagrant file"
      stage.vm.box = "dummy"
      stage.vm.box_url = "https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box"

      stage.vm.synced_folder '.', ENV['BASE_FOLDER'],
        disabled: false,
        type: 'rsync'

      stage.vm.provider :aws do |aws,override|
        aws.keypair_name = ENV['STAGE_KEYPAIR_NAME']
        aws.ami = ENV['STAGE_AMI']
        aws.instance_type = ENV['STAGE_INSTANCE_TYPE']
        aws.region = ENV['STAGE_REGION']
        aws.subnet_id = ENV['STAGE_SUBNET_ID']
        aws.security_groups = ENV['STAGE_SECURITY_GROUPS']
        aws.associate_public_ip = true
        aws.iam_instance_profile_name = ENV['STAGE_INSTANCE_PROFILE_NAME']
        aws.aws_dir = ENV['HOME'] + "/.aws/"
        aws.aws_profile = "#{ENV['STAGE_AWS_PROFILE']}"

        override.ssh.username = "ubuntu"
  #			override.ssh.private_key_path = ENV['STAGE_SSH_PRIVATE_KEY_PATH']
      end

    end
  end
  if chosen_environment == "prod"
    config.vm.define "prod" do |prod|

      puts "Inside config.vm.define Prod Vagrant file"
      prod.vm.box = "dummy"
      prod.vm.box_url = "https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box"

      prod.vm.synced_folder '.', ENV['BASE_FOLDER'],
        disabled: false,
        type: 'rsync'

      prod.vm.provider :aws do |aws,override|
        aws.keypair_name = ENV['PROD_KEYPAIR_NAME']
        aws.ami = ENV['PROD_AMI']
        aws.instance_type = ENV['PROD_INSTANCE_TYPE']
        aws.elastic_ip = ENV['PROD_ELASTIC_IP']
        aws.region = ENV['PROD_REGION']
        aws.subnet_id = ENV['PROD_SUBNET_ID']
        aws.security_groups = ENV['PROD_SECURITY_GROUPS']
        aws.associate_public_ip = true
        aws.iam_instance_profile_name = ENV['PROD_INSTANCE_PROFILE_NAME']
        aws.aws_dir = ENV['HOME'] + "/.aws/"
        aws.aws_profile = "#{ENV['PROD_AWS_PROFILE']}"

        override.ssh.username = "ubuntu"
        override.ssh.private_key_path = ENV['PROD_SSH_PRIVATE_KEY_PATH']
      end

    end
  end
end
