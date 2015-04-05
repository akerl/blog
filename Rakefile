require 'rspec/core/rake_task'
require 'rubocop/rake_task'
require 'link_checker'

desc 'Run tests'
RSpec::Core::RakeTask.new(:spec)

desc 'Run Rubocop on the code'
RuboCop::RakeTask.new(:rubocop) do |task|
  task.patterns = ['spec/**/*.rb']
  task.fail_on_error = true
end

desc 'Build the site'
task :build do
  fail unless system 'jekyll build'
end

desc 'Check all external links'
task :check_links do
  LinkChecker.new(target: '_build').check_uris
end

task default: [:spec, :rubocop, :build, :check_links]
