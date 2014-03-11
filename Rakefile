require 'rspec/core/rake_task'
require 'rubocop/rake_task'

desc 'Run tests'
RSpec::Core::RakeTask.new(:spec)

desc 'Run Rubocop on the gem'
Rubocop::RakeTask.new(:rubocop) do |task|
  task.patterns = ['spec/**/*.rb']
  task.fail_on_error = true
end

desc 'Run travis-lint on .travis.yml'
task :travislint do
  print 'There is an issue with your .travis.yml' unless system('travis-lint')
end

desc 'Build the site'
task :build do
  `jekyll build`
end

task default: [:spec, :travislint, :rubocop, :build]
