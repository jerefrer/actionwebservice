require 'rubygems'
require 'rake'
require 'rake/testtask'
require 'rake/rdoctask'
require 'rake/packagetask'
require 'rake/gempackagetask'
require 'rake/contrib/rubyforgepublisher'
require 'fileutils'
require File.join(File.dirname(__FILE__), 'lib', 'action_web_service', 'version')

PKG_BUILD     = ENV['PKG_BUILD'] ? '.' + ENV['PKG_BUILD'] : ''
PKG_NAME      = 'actionwebservice'
PKG_VERSION   = ActionWebService::VERSION::STRING + PKG_BUILD
PKG_FILE_NAME   = "#{PKG_NAME}-#{PKG_VERSION}"
PKG_DESTINATION = ENV["RAILS_PKG_DESTINATION"] || "../#{PKG_NAME}"

RELEASE_NAME  = "REL #{PKG_VERSION}"

RUBY_FORGE_PROJECT = "aws"
RUBY_FORGE_USER    = "webster132"

desc "Default Task"
task :default => [ :test ]


# Run the unit tests
Rake::TestTask.new { |t|
  t.libs << "test"
  t.test_files = Dir['test/*_test.rb']
  t.verbose = true
}

SCHEMA_PATH = File.join(File.dirname(__FILE__), *%w(test fixtures db_definitions))

desc 'Build the MySQL test database'
task :build_database do 
  %x( mysqladmin -uroot create actionwebservice_unittest )
  %x( mysql -uroot actionwebservice_unittest < #{File.join(SCHEMA_PATH, 'mysql.sql')} )
end

desc 'Build the sqlite3 test database'
task :build_sqlite3_database do
  filename = 'actionwebservice_unittest.db'
  File.delete filename if File.exist? filename
  %x(sqlite3 #{filename}  < #{File.join(SCHEMA_PATH, 'sqlite3.sql')})
end


# Generate the RDoc documentation
Rake::RDocTask.new { |rdoc|
  rdoc.rdoc_dir = 'doc'
  rdoc.title    = "Action Web Service -- Web services for Action Pack"
  rdoc.options << '--line-numbers' << '--inline-source'
  rdoc.options << '--charset' << 'utf-8'
  rdoc.template = "#{ENV['template']}.rb" if ENV['template']
  rdoc.rdoc_files.include('README')
  rdoc.rdoc_files.include('CHANGELOG')
  rdoc.rdoc_files.include('lib/action_web_service.rb')
  rdoc.rdoc_files.include('lib/action_web_service/*.rb')
  rdoc.rdoc_files.include('lib/action_web_service/api/*.rb')
  rdoc.rdoc_files.include('lib/action_web_service/client/*.rb')
  rdoc.rdoc_files.include('lib/action_web_service/container/*.rb')
  rdoc.rdoc_files.include('lib/action_web_service/dispatcher/*.rb')
  rdoc.rdoc_files.include('lib/action_web_service/protocol/*.rb')
  rdoc.rdoc_files.include('lib/action_web_service/support/*.rb')
}


# Create compressed packages
spec = Gem::Specification.new do |s|
  s.platform = Gem::Platform::RUBY
  s.name = PKG_NAME
  s.summary = "Web service support for Action Pack."
  s.description = %q{Adds WSDL/SOAP and XML-RPC web service support to Action Pack}
  s.version = PKG_VERSION

  s.author = "Leon Breedt, Kent Sibilev"
  s.email = "bitserf@gmail.com, ksibilev@yahoo.com"
  s.rubyforge_project = "aws"
  s.homepage = "http://www.rubyonrails.org"

  s.add_dependency('actionpack', '= 2.3.14' + PKG_BUILD)
  s.add_dependency('activerecord', '= 2.3.14' + PKG_BUILD)

  s.has_rdoc = true
  s.requirements << 'none'
  s.require_path = 'lib'
  s.autorequire = 'actionwebservice'

  s.files = [ "Rakefile", "setup.rb", "README", "TODO", "CHANGELOG", "MIT-LICENSE" ]
  s.files = s.files + Dir.glob( "examples/**/*" ).delete_if { |item| item.match( /\.(svn|git)/ ) }
  s.files = s.files + Dir.glob( "lib/**/*" ).delete_if { |item| item.match( /\.(svn|git)/ ) }
  s.files = s.files + Dir.glob( "test/**/*" ).delete_if { |item| item.match( /\.(svn|git)/ ) }
  s.files = s.files + Dir.glob( "generators/**/*" ).delete_if { |item| item.match( /\.(svn|git)/ ) }
end
Rake::GemPackageTask.new(spec) do |p|
  p.gem_spec = spec
  p.need_tar = true
  p.need_zip = true
end


# Publish beta gem
desc "Publish the API documentation"
task :pgem => [:package] do 
  Rake::SshFilePublisher.new("davidhh@wrath.rubyonrails.org", "public_html/gems/gems", "pkg", "#{PKG_FILE_NAME}.gem").upload
  `ssh davidhh@wrath.rubyonrails.org './gemupdate.sh'`
end

# Publish documentation
desc "Publish the API documentation"
task :pdoc => [:rdoc] do 
  Rake::SshDirPublisher.new("davidhh@wrath.rubyonrails.org", "public_html/aws", "doc").upload
end


def each_source_file(*args)
	prefix, includes, excludes, open_file = args
	prefix ||= File.dirname(__FILE__)
	open_file = true if open_file.nil?
	includes ||= %w[lib\/action_web_service\.rb$ lib\/action_web_service\/.*\.rb$]
	excludes ||= %w[lib\/action_web_service\/vendor]
	Find.find(prefix) do |file_name|
		next if file_name =~ /\.svn/
		file_name.gsub!(/^\.\//, '')
		continue = false
		includes.each do |inc|
			if file_name.match(/#{inc}/)
				continue = true
				break
			end
		end
		next unless continue
		excludes.each do |exc|
			if file_name.match(/#{exc}/)
				continue = false
				break
			end
		end
		next unless continue
		if open_file
			File.open(file_name) do |f|
				yield file_name, f
			end
		else
			yield file_name
		end
	end
end

desc "Count lines of the AWS source code"
task :lines do
  total_lines = total_loc = 0
  puts "Per File:"
	each_source_file do |file_name, f|
    file_lines = file_loc = 0
    while line = f.gets
      file_lines += 1
      next if line =~ /^\s*$/
      next if line =~ /^\s*#/
      file_loc += 1
    end
    puts "  #{file_name}: Lines #{file_lines}, LOC #{file_loc}"
    total_lines += file_lines
    total_loc += file_loc
  end
  puts "Total:"
  puts "  Lines #{total_lines}, LOC #{total_loc}"
end

desc "Publish the release files to RubyForge."
task :release => [ :package ] do
  require 'rubyforge'

  packages = %w( gem tgz zip ).collect{ |ext| "pkg/#{PKG_NAME}-#{PKG_VERSION}.#{ext}" }

  rubyforge = RubyForge.new
  rubyforge.login
  rubyforge.add_release(PKG_NAME, PKG_NAME, "REL #{PKG_VERSION}", *packages)
end
