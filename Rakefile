require 'rugged'
require 'fileutils'
require 'yaml'
require 'open3'
require 'erb'
require 'highline/import'
require 'tempfile'

def skip_path?(file)
  [".bundle", ".git", "vendor/gems", "puppet" ].each do |dir|
    return true if file.match(dir)
  end
  return false
end

def render_template(template, output, scope)
  tmpl = File.read(template)
  erb = ERB.new(tmpl, 0, "<>")
  begin
    File.open(output, "a+") do |f|
        f.puts erb.result(scope)
    end
  rescue Exception => e
    puts ''
    puts e.message
    exit 1
  end
end

# Global Variables
clonedir = './puppet/environments/masterless'
control_repo = 'https://github.com/smbambling/r2_puppet_control.git'

desc "Clone Puppet Control Repository AND Run R10k"
#task :default => [:'pcr:clone', :'r10k:install', :'eyaml:decrypt_domains' , 'hieraconfig:hieraconf_update']
task :default => [:'pcr:clone', :'r10k:install']

namespace :pcr do
  desc "Clone Puppet Control Repository"
  task :clone do
    if ! File.exists?("#{clonedir}/.git/config")
      branch = ask("What branch of the control_repository do you want to clone ? : ") { |q| q.default = "master" }
      print "Cloning Puppet Control Repository..."
      begin
        Rugged::Repository.clone_at("#{control_repo}", "#{clonedir}", {
          :checkout_branch => branch
          }
        )
      rescue Exception => e
        puts ''
        puts e.message
        exit 1
      end
      print "Done\n"
    end
  end

  desc "Update Puppet Control Repository (git pull)"
  task :update do
    if File.exists?("#{clonedir}/.git/config")
      print "Updating Puppet Control Repository..."
      begin
        # http://www.synbioz.com/blog/rugged_gem_git
        repo = Rugged::Repository.new("#{clonedir}")
        branch_name = repo.head.name.sub(/^refs\/heads\//, '')
        repo.fetch('origin', [repo.head.name])
        origin_commit = repo.branches["origin/#{branch_name}"].target
        repo.references.update(repo.head, origin_commit.oid)
      rescue Exception => e
        puts ''
        puts e.message
        exit 1
      end
      print "Done\n"
    end
  end

  desc "Clean Puppet Control Repository"
  task :clean do
    def get_yn_input
      input = ''
      STDOUT.flush
      input = STDIN.gets.chomp
      case input.upcase
      when "Y"
        if File.directory?("./puppet")
          begin
            print "Removing ./puppet directory..."
            FileUtils.rm_rf("./puppet")
          rescue Exception => e
            puts ''
            puts e.message
            exit 1
          end
          print "Done\n"
          Rake::Task["default"].invoke
        end
      when "N"
        puts "Aborting Tasks"
      else
        puts "Please enter Y or N"
        get_yn_input
      end
    end
    puts "This will delete any modification in #{clonedir}"
    print "Do you want to continue ? "
    get_yn_input
  end
end

namespace :r10k do
  desc "Run R10k to fetch modules in Pupetfile"
  task :install do
    print "Running R10k to fetch modules..."
    begin
      system("bundle exec r10k puppetfile install \
        --puppetfile #{clonedir}/Puppetfile \
        --moduledir #{clonedir}/modules"
      )
    rescue Exception => e
      puts ''
      puts e.message
      exit 1
    end
    print "Done\n"
  end
end

namespace :eyaml do
  desc "decrypt domain yaml files"
  task :decrypt_domains do
    print "Decrypting domain hiera..."
    begin
      gpg_key_path = ask("Where is your gpg keyring? : ") { |q| q.default = "~/ARIN/trap/puppet_keyring" }
      yaml_path = "./puppet/environments/masterless/hieradata"
      domain_yamls = Dir["#{yaml_path}/domain/*.yaml"]
      domain_yamls.each do |domain|
        system("bundle exec eyaml decrypt -f \
               #{domain} \
               --gpg-gnupghome=#{gpg_key_path} \
              --gpg-recipients='puppet@arin.net' > #{domain}_decrypted"
              )
        FileUtils.mv("#{domain}_decrypted", "#{domain}")
      end
    rescue Execption => e
      puts ''
      puts e.message
      exit 1
    end
    print "Done\n"
  end

  desc "decrypt users yaml file"
  task :decrypt_users do
    print "Decrypting users hiera..."
    begin
      gpg_key_path = ask("Where is your gpg keyring? : ") { |q| q.default = "~/ARIN/trap/puppet_keyring" }
      yaml_path = "./puppet/environments/masterless/hieradata"
      user_yaml = Dir["#{yaml_path}/accounts/users.yaml"]
      system("bundle exec eyaml decrypt -f \
             #{user_yaml} \
             --gpg-gnupghome=#{gpg_key_path} \
            --gpg-recipients='puppet@arin.net' > #{user}_decrypted"
            )
      FileUtils.mv("#{user}_decrypted", "#{user_yaml}")
    rescue Execption => e
      puts ''
      puts e.message
      exit 1
    end
    print "Done\n"
  end
end

namespace :hieraconfig do
  desc "Update hiera to point to the proper datadir"
  task :hieraconf_update do
    print "updating hiera.yaml"
    begin
      filename = 'puppet/environments/masterless/hiera.yaml'
      reg_pattern = /\/vagrant\/puppet\/environments\/dev\/hieradata\//
      reg_replace = "/tmp/puppet/environments/masterless/hieradata"
      File.open(filename = filename, "r+") do |file|
        file << File.read(filename).gsub(reg_pattern, reg_replace)
      end
    rescue Exception => e
      puts ''
      puts e.message
      exit 1
    end
  end
end
