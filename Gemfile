path = File.dirname(ENV['BUNDLE_GEMFILE'] || '.')
ruby File.read(File.join(path, '.circle-ruby')).chomp
source 'https://rubygems.org'

gem 'html-proofer', '~> 3.7.0'
gem 'jekyll', '~> 3.6.0'
gem 'kramdown', '~> 1.16.0'
gem 'sass', '~> 3.5.0'
gem 'therubyracer', '~> 0.12.3'
gem 'uglifier', '~> 4.0.1'

group :jekyll_plugins do
  gem 'jekyll-assets', '~> 3.0.1'
  gem 'jekyll-gist', '~> 1.5.0'
end

group :development do
  gem 'fuubar', '~> 2.2.0'
  gem 'goodcop', '~> 0.1.1'
  gem 'rake', '~> 12.3.0'
  gem 'rspec', '~> 3.7.0'
  gem 'rubocop', '~> 0.52.0'
end
