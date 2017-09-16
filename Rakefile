require 'rspec/core/rake_task'
require 'rubocop/rake_task'
require 'html-proofer'

desc 'Run tests'
RSpec::Core::RakeTask.new(:spec)

desc 'Run Rubocop on the code'
RuboCop::RakeTask.new(:rubocop) do |task|
  task.fail_on_error = true
end

desc 'Build the site'
task :build do
  raise unless system 'jekyll build'
end

desc 'Check rendered site'
task :test do
  HTMLProofer.check_directory(
    './_build',
    check_html: true,
    validation: {
      report_missing_names: true,
      report_script_embeds: true
    },
    typhoeus: {
      ssl_verifypeer: false
    }
  ).run
end

task default: %i[spec rubocop build test]
